---
title: Global Server Load Balancing with NS1 and NGINX Plus
description: Set up global server load balancing with IBM NS1 Connect for domains that F5 NGINX Plus serves, with the NGINX NS1 agent reporting load and health.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "GSLB, global server load balancing, NS1, IBM NS1 Connect, data feed, filter chain, shed load, NGINX Plus API"
f5-summary: >
  This guide shows how to set up global server load balancing (GSLB) with NS1 for three F5 NGINX Plus points of presence.
  An agent reports the health and load of each NGINX Plus instance to NS1, which then sends each client to a working point of presence.
  F5 archived the agent in 2023, so the guide is for reference and testing.
f5-audience: operator
---

## Overview

Global server load balancing (GSLB) refers to the intelligent distribution of traffic across server resources located in multiple points of presence (PoPs). GSLB is most commonly implemented by controlling the responses to DNS requests, directing each user to the most appropriate destination IP address based on the availability, performance, and proximity of each PoP.

Many DNS providers offer some form of GSLB. [NS1](https://www.ibm.com/products/ns1-connect) has one of the most advanced solutions available as a service, with a rich API that PoPs can use to dynamically inform the NS1 servers about their availability and current loads.

This document describes how to use NGINX’s NS1 agent to interface with the NS1 service, enabling sophisticated GSLB across multiple PoPs that are powered by F5 NGINX Plus. You can colocate an instance of the agent alongside each NGINX Plus instance, or configure an agent to query one or more NGINX Plus instances remotely. (This guide covers only the colocation option.)

The agent periodically queries the NGINX Plus API for several metrics that it uses to calculate the current number of active connections and load average on the NGINX Plus instance, and reports those metrics to NS1. The agent can also be configured to report the status of the site as ``up:`` ``true`` or ``up:`` ``false`` (that is, down).

The agent supports the following capabilities:

- Remote health checks, so clients are not directed to an unavailable (down or otherwise unreachable) PoP
- Local capacity checks, so clients are not directed to a PoP without enough healthy servers
- Central capacity balancing, so clients are balanced across PoPs according to the current load at each PoP, and traffic is drained from PoPs that are overloaded

{{< call-out class="caution" >}} F5 archived the [nginx-ns1-gslb](https://github.com/nginxinc/nginx-ns1-gslb) agent repository in 2023, and no longer maintains the agent. Its last release is 0.1.1. NS1 is now IBM NS1 Connect, so its console and documentation can differ from this guide. {{< /call-out >}}

The solution functions alongside other NS1 capabilities, such as geo-proximal routing which directs each client to the closest PoP.

<span id="about-nginx"></span>

### About NGINX Plus

[NGINX Plus](https://www.f5.com/products/nginx/nginx-plus) adds features to NGINX Open Source, such as active health checks and the [NGINX Plus API]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}}), which the agent reads.

<span id="prereqs"></span><span id="prerequisites"></span>

## Before you begin

Before you begin, make sure you have:

- **A registered domain name**: You delegate it to NS1.
- **An NS1 account**: NS1 is now [IBM NS1 Connect](https://www.ibm.com/products/ns1-connect).
- **Three or more NGINX Plus instances**: Each one needs the NGINX Plus API, as described in [Configure the API]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md#json_configure" >}}).
- **Go 1.19 or later on each instance**: The agent's `go.mod` requires it. To install Go, see [Download and install](https://go.dev/doc/install). With Docker, you can use the agent's container image instead.

<span id="ns1-setup"></span><span id="setting-up-ns1"></span>

## Set up NS1

1. Log in to your NS1 account, select **ZONES** in the title bar to open the **Your Zones** page, and select the **+ Add Zone** button in the upper right corner.

   <img src="/nginx/images/ns1-zones-tab.png" alt="Screenshot of NS1 GUI: ZONES tab" width="1024" height="442" class="aligncenter size-full wp-image-63029" />

2. In the **Add Zone** pop-up window, enter the domain name (**nginxgslb.cf** in this guide) in the **Domain Name** field. We're not changing any of the default settings, but see the [NS1 documentation](https://www.ibm.com/docs/en/ns1-connect) for information about TTL (time-to-live) settings. Select the **Save Zone** button.

   <img src="/nginx/images/ns1-add-zone-popup.png" alt="Screenshot of NS1 GUI: Add Zone popup" width="612" height="556" class="aligncenter size-full wp-image-63028" />

3. On the page that opens, select the **NAMESERVERS** tab and follow [these instructions](https://www.ibm.com/docs/en/ns1-connect) to delegate the new domain name to NS1.

   <img src="/nginx/images/ns1-nameservers-page.png" alt="Screenshot of NS1 GUI: NAMESERVERS page" width="1024" height="443" class="aligncenter size-full wp-image-63027" />

4. Select the **RECORDS** tab. As shown in the screenshot, an ``NS`` (Name Server) record has already been created automatically and appears in the white box. Select either **Add Record** button.

   <img src="/nginx/images/ns1-records-page-ns.png" alt="Screenshot of NS1 GUI: RECORDS page" width="1024" height="948" class="aligncenter size-full wp-image-63026" />

5. The **Add Record** window pops up. Enter the following values:

   - **Record Type** – **A** (the default).
   - **name** – Leave blank unless you are creating the ``A`` record for a subdomain.
   - **TTL** – **3600** is the default, which we are not changing.
   - **ANSWERS** – The public IP address of the first NGINX Plus instance. To add each of the other instances, select the **Add Answer** button. (In this guide we're using private IP addresses in the 10.0.0.0/8 range as examples.)

   Select the **Save All Changes** button.

   <img src="/nginx/images/ns1-add-record-popup.png" alt="Screenshot of NS1 GUI: Add Record popup" width="612" height="882" class="aligncenter size-full wp-image-63025" />

6. The new ``A`` record appears on the **RECORDS** tab. Select the **3 Answers** button at the right end of its row to open the details page for the ``A`` record. (The screenshot – like subsequent screenshots of this page and the details page – shows only the bottom half of the tab.)

   <img src="/nginx/images/ns1-2-records.png" alt="Screenshot of NS1 GUI: NS and A records for nginxgslb.cf" width="1024" height="377" class="aligncenter size-full wp-image-63024" />

7. The window that opens shows details for the ``A`` record. The IP addresses of the NGINX Plus instances appear in the **Ungrouped Answers** section. Select the stacked dots icon at the right end of the field for the first address (10.10.10.1 in this guide) and select **Edit Answer Metadata**.

   <img src="/nginx/images/ns1-3-answers.png" alt="Screenshot of NS1 GUI: list of answers for nginxgslb.cf" width="1024" height="393" class="aligncenter size-full wp-image-63023" />

8. In the **Answer Metadata** window that pops up, select **Up/down** in the **STATUS** section of the **SETTING** column, if it is not already selected. Select the **Select** box in the **AVAILABLE** column, and then select either **Up** or **Down** from the drop-down menu. In this guide we're selecting **Up** to indicate that the NGINX Plus instance is operational.
   <img src="/nginx/images/ns1-answer-metadata-status.png" alt="Screenshot of NS1 GUI: setting STATUS answer metadata" width="1024" height="1105" class="aligncenter size-full wp-image-63022" />

9. Select a value in the **GEOGRAPHICAL** section of the **SETTING** column and specify the location of the NGINX Plus instance. Begin by choosing one of the several types of codes that NS1 offers for identifying locations:

   - **Canadian province(s)** – Two-letter codes for Canadian provinces
   - **Country/countries** – Two-letter codes for nations and territories
   - **Geographic region(s)** – Identifiers like **US-WEST** and **ASIAPAC**
   - **ISO region code** – Identification codes for nations and territories as defined in [ISO 3166](https://www.iso.org/iso-3166-country-codes.html)
   - **Latitude** – Degrees, minutes, and seconds of latitude (northern or southern hemisphere)
   - **Longitude** – Degrees, minutes, and seconds of longitude (eastern or western hemisphere)
   - **US State(s)** – Two-letter codes for US states

   In this guide we're using **Country/countries** codes. For the first NGINX Plus instance, we select **Americas > Northern America > United States (US)** and select the **Ok** button.

    <img src="/nginx/images/ns1-answer-metadata-geographical.png" alt="Screenshot of NS1 GUI: setting GEOGRAPHICAL answer metadata" width="1024" height="1024" class="aligncenter size-full wp-image-63021" />

10. Repeat Steps 7–9 for both of the other two NGINX Plus instances. For the country in Step 9, we're selecting **Europe > Western Europe > Germany (DE)** for NGINX Plus instance 2 and **Asia > South-Eastern Asia > Singapore (SG)** for NGINX Plus instance 3.

    When finished with both instances, on the details page for the ``A`` record select the **Save Record** button.

11. Select the **Create Filter Chain** button to create a filter chain based on the **Up/Down** and **Country/countries** metadata (for an overview of filter chains, see the [NS1 documentation](https://www.ibm.com/docs/en/ns1-connect)).

    <img src="/nginx/images/ns1-create-filter-chain.png" alt="Screenshot of NS1 GUI: creating filter chain" width="1024" height="538" class="aligncenter size-full wp-image-63020" />

12. In the **Add Filters** window that pops up, select the plus sign (+) on the button for each filter you want to apply. In this guide, we're configuring the filters in this order:

    - **Up** in the ** HEALTHCHECKS ** section
    - **Geotarget Country** in the ** GEOGRAPHIC ** section
    - **Select First N** in the ** TRAFFIC MANAGEMENT ** section

    Select the **Save Filter Chain**  button.

    <img src="/nginx/images/ns1-add-filters-summary.png" alt="Screenshot of NS1 GUI: Add Filters page with three filters defined" width="1024" height="543" class="aligncenter size-full wp-image-63019" />

<span id="agent-install"></span><span id="installing-the-ns1-agent"></span>

## Install the NS1 agent

In this section we install and configure the NS1 agent on the same hosts as our NGINX Plus instances. We also create an NS1 _data feed_ for each agent, which enables it to send status information about its paired NGINX Plus instance to NS1 via the NS1 API.

1. Follow the instructions in the [NS1 documentation](https://www.ibm.com/docs/en/ns1-connect) to set up and connect a separate data feed for each of the three NGINX Plus instances, which NS1 calls _answers_.

   On the first page (**Configure a new data source from NSONE Data Feed API v1**) specify a name for the _data source_, which is the administrative container for the data feeds you will be creating. Use the same name each of the three times you go through the instructions. We're naming the data source **NGINX-GSLB**.

   On the next page (**Create Feed from NSONE Data Feed API v1**), create a data feed for the instance. Because the **Name** field is just for internal use, any value is fine. The value in the **Label** field is used in the YAML configuration file for the instance (see Step 4 below). We're specifying labels that indicate the country (using the ISO 3166 codes) in which the instance is running:

   - **us-nginxgslb-datafeed** for instance 1 in the US
   - **de-nginxgslb-datafeed** for instance 2 in Germany
   - **sg-nginxgslb-datafeed** for instance 3 in Singapore

   After creating the three feeds, note the value in the **Feeds URL** field on the **INTEGRATIONS** tab. The final element of the URL is the ``<NS1_DATA_SOURCE_ID>`` you will specify in the YAML configuration file in Step 4.

2. Follow the instructions in the [NS1 documentation](https://www.ibm.com/docs/en/ns1-connect) to create an NS1 API key for the agent, if you have not already. To open **Account Settings**, select your user name in the upper right corner of the NS1 title bar. We're naming the app **NGINX-GSLB**. Make note of the key value – you'll specify it as ``<NS1_API_KEY>`` in the YAML configuration file in Step 4. To see the actual hexadecimal value, select the circled letter **i** in the **API Key** field.

3. On each NGINX Plus host, clone the [GitHub repo](https://github.com/nginxinc/nginx-ns1-gslb) for the NS1 agent.

4. On each NGINX Plus host, create the [YAML configuration file](https://github.com/nginxinc/nginx-ns1-gslb/blob/master/configs/README.md) for the NS1 agent. In this guide we're using the following file:

   ```none
   agent:
     interval: 10
     retry_time: 5
   nginx_plus:
     hosts:
       - host: "127.0.0.1:8000"
         resolve: false
         host_header: "127.0.0.1:8000"
     api_endpoint: "/api"
     client_timeout: 10
   nsone:
     api_key: "<NS1_API_KEY>"
     client_timeout: 10
     source_id: "<NS1_DATA_SOURCE_ID>"
   services:
     method: "upstream_groups"
     threshold: 2
     sampling_type: "count"
     feeds:
       - name: "my_backend"
         feed_name: "<xx>-nginxgslb-datafeed"
   ```

   The ``hosts`` section configures the agent to run on the same host as the NGINX Plus instance from which it collects metrics – in this guide, localhost. Because localhost is identified by its IP address (127.0.0.1) in the ``host`` field, hostname resolution is unnecessary and ``resolve`` is set to ``false``. The agent gathers metrics from the NGINX Plus API (the ``/api`` endpoint) on port 8000.

   In the ``nsone`` section, include the ``<NS1_API_KEY>`` and ``<NS1_DATA_SOURCE_ID>`` values you noted in Step 2 and Step 1, respectively.

   In the ``services`` section, we're specifying [upstream_groups](https://github.com/nginxinc/nginx-ns1-gslb/tree/master/configs#services) as the method for the NS1 agent to use, meaning that it collects metrics about the upstream group that the NGINX Plus instance is load balancing – ``my_backend``, as specified in the ``name`` field of the ``feeds`` section. The ``threshold`` field defines how many servers in the upstream group must be healthy for the backend app to be considered up, and the ``sampling_type`` field tells the agent to collect the sum of active connections to backend servers. (We're leaving actual setup of the backend app and the configuration file for the NGINX Plus instance as exercises for the reader.)

   The agent configuration is the same for each of the paired agents and NGINX Plus instances, except for the value in the ``feed_name`` field (see Step 1 for the feed names). If you choose to configure a different upstream group for each instance, also change the value in the ``name`` field.

   For details about all fields in the configuration file, see the documentation in our [GitHub repo](https://github.com/nginxinc/nginx-ns1-gslb/blob/master/configs/README.md).

5. Start the agent in the directory of the cloned repository. Replace `<CONFIG_FILE>` with the path of your configuration file:

   ```shell
   go run cmd/agent/main.go --config-file <CONFIG_FILE>
   ```

   To run the agent in a container instead, use its last image, and mount your configuration file. Give its absolute path. The `--network host` option lets the agent reach the NGINX Plus API at `127.0.0.1:8000` of the host:

   ```shell
   docker run --network host -v <CONFIG_FILE>:/etc/nginx-ns1-gslb/config.yaml ghcr.io/nginxinc/nginx-ns1-gslb:0.1.1
   ```

   For details, see [Running the agent](https://github.com/nginxinc/nginx-ns1-gslb#running-the-agent).

<span id="verify-redistribution"></span><span id="verifying-that-ns1-redistributes-traffic"></span>

## Verify that NS1 redistributes traffic

In this section we describe how to verify that NS1 correctly redistributes traffic to an alternate PoP when the PoP nearest to the client is not operational (in the setup in this guide, each of the three NGINX Plus instances corresponds to a PoP). There are three ways to indicate to NS1 that a PoP is down:

- [Change the status of the NGINX Plus instance](#verify-when-status-down) to **Down** in the NS1 ``A`` record
- [Take down the servers in the proxied upstream group](#verify-when-upstream-down)
- [Cause traffic to exceed a configured threshold](#verify-when-over-threshold)

<span id="verify-when-status-down"></span><span id="verifying-traffic-redistribution-when-an-nginxplus-instance-is-marked-down"></span>

### Mark an NGINX Plus instance as down

Here we verify that NS1 switches over to the next-nearest NGINX Plus instance when we change the metadata on the nearest NGINX Plus instance to **Down**.

1. On a host located in the US, run the following command to determine which site NS1 is returning as nearest. Appropriately, it's returning 10.10.10.1, the IP address of the NGINX Plus instance in the US.

   ```shell
   $ nslookup nginxgslb.cf

   Server:     10.10.100.102
   Address:    10.10.100.102#53

   Non-authoritative answer:
   Name:	nginxgslb.cf
   Address: 10.10.10.1
   ```

2. Change the **Up/Down** answer metadata on the US instance to **Down** (see Step 8 in [Set up NS1](#ns1-setup)).

3. Wait an hour – because we didn't change the default time-to-live (TTL) of 3600 seconds on the ``A`` record for **nginxgslb.cf** – and issue the ``nslookup`` command again. NS1 returns 10.10.10.2, the IP address of the NGINX Plus instance in Germany, which is now the nearest.

   ```shell
   $ nslookup nginxgslb.cf

   Server:     10.10.100.102
   Address:    10.10.100.102#53

   Non-authoritative answer:
   Name:	nginxgslb.cf
   Address: 10.10.10.2
   ```

<span id="verify-when-upstream-down"></span><span id="verifying-traffic-redistribution-when-an-upstream-group-is-down"></span>

### Take down an upstream group

With our NGINX Plus instances (answers) connected to NS1 data feeds (see Step 2 in [Install the NS1 agent](#agent-install)), we can verify that NS1 redistributes traffic correctly when it receives data from the agent indicating that an upstream group is down. In the following example, it's the **my_backend** group that goes down, as reported in the **us-nginxgslb-datafeed** feed.

We run the following commands on a host located in the US.

1. Query the NGINX Plus API to verify that the current status is ``up`` for the **my_backend** upstream group being proxied by the NGINX Plus instance in the US:

   ```shell
   $ curl -X GET "127.0.0.1:8000/api/<VERSION>/http/upstreams/my_backend/" -H "accept: application/json" | python3 -m json.tool | grep state

   "state": "up",
   ```

   Replace `<VERSION>` with the version of the NGINX Plus API.

2. Query the NS1 API to verify that NS1 also sees the US **my_backend** upstream group as ``up``. (For details about this API call, see the [NS1 documentation](https://developer.ibm.com/apis/catalog/ns1--ibm-ns1-connect-api/Introduction). If the page doesn't scroll automatically to the relevant section, search for "Get active data feeds for a source".)

   On the command line, ``<NS1_API_KEY>`` and ``<NS1_DATA_SOURCE_ID>`` are the same values we included in the YAML file in Step 4 of [Install the NS1 agent](#agent-install).

   The output includes a ``destinations`` entry for each data feed, so we search for the one where the ``label`` field says ``us-nginxgslb-datafeed``, and verify that the ``up`` field in that entry says ``true``.

    ```shell
   $ curl -X GET -H 'X-NSONE-Key: <NS1_API_KEY>' https://api.nsone.net/v1/data/feeds/<NS1_DATA_SOURCE_ID> | python3 -m json.tool
   [
     ...
     {
       "destinations": [
         {
           "destid": "<ANSWER_ID>",
           "desttype": "answer",
           "record": "<RECORD_ID>"
         }
       ],
       "id": "<FEED_ID>",
       "data": {
         "up": "true"
       },
       "config": {
         "label": "us-nginxgslb-datafeed"
       },
       "name": "us-nginxgslb-datafeed"
     },
     ...
   ]
   ```

    Where `<ANSWER_ID>` is the ID of the answer in your NS1 record and `<RECORD_ID>` is the ID of your NS1 record.

3. Determine which site NS1 is returning for hosts in the US. Appropriately, it's 10.10.10.1, the IP address of the US-based NGINX Plus instance.

   ```shell
   $ nslookup nginxgslb.cf

   Server:     10.10.100.102
   Address:    10.10.100.102#53

   Non-authoritative answer:
   Name:	nginxgslb.cf
   Address: 10.10.10.1
   ```

4. Take down the servers in the **my_backend** upstream group. There are several ways to do this: turn off the actual app on each server; or on the NGINX Plus instance either change the app's port number in the NGINX Plus configuration file to the wrong value, or use the [server](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) directive or the [NGINX Plus API](https://docs.nginx.com/nginx/admin-guide/load-balancer/dynamic-configuration-api/#using-the-api-for-dynamic-configuration) to set each server's state to ``down``.

5. Repeat Step 1. The NGINX Plus API now reports the status as ``unhealthy``. This status comes from active health checks. If you mark the servers as down through the API instead, the status is ``down``.

   ```none
   $ curl -X GET "127.0.0.1:8000/api/<VERSION>/http/upstreams/my_backend/" -H "accept: application/json" | python3 -m json.tool | grep state

   "state": "unhealthy",
   ```

6. Repeat Step 2. The NS1 API now returns ``false`` in the ``up`` field.

   ```shell
   $ curl -X GET -H 'X-NSONE-Key: <NS1_API_KEY>' https://api.nsone.net/v1/data/feeds/<NS1_DATA_SOURCE_ID> | python3 -m json.tool
   [
     ...
     {
       "destinations": [
         {
           "destid": "<ANSWER_ID>",
           "desttype": "answer",
           "record": "<RECORD_ID>"
         }
       ],
       "id": "<FEED_ID>",
       "data": {
         "up": "false"
       },
       "config": {
         "label": "us-nginxgslb-datafeed"
       },
       "name": "us-nginxgslb-datafeed"
     },
     ...
   ]
   ```

7. Wait an hour – because we didn't change the default TTL of 3600 seconds on the ``A`` record for **nginxgslb.cf** – and repeat Step 3. NS1 returns 10.10.10.2, the IP address of the NGINX Plus instance in Germany, which is now the nearest to US-based hosts.

   ```shell
   $ nslookup nginxgslb.cf

   Server:     10.10.100.102
   Address:    10.10.100.102#53

   Non-authoritative answer:
   Name:	nginxgslb.cf
   Address: 10.10.10.2
   ```

<span id="verify-when-over-threshold"></span><span id="verifying-traffic-redistribution-when-a-threshold-is-exceeded"></span>

### Exceed a load threshold

You can configure NS1 to redistribute traffic away from a given NGINX Plus instance when a load metric for the instance exceeds one or more thresholds that you set. The thresholds are set in an NS1 _shed filter_, so-called because NS1 describes the shifting of traffic to a different IP address as "shedding load" from the current IP address.

Here we verify that NS1 redistributes traffic correctly when the number of active connections on an instance exceeds the threshold we set.

<span id="creating-the-shed-filter"></span>

#### Create the shed filter

First we perform these steps to create the shed filter:

1. Navigate to the details page of the ``A`` record for **nginxgslb.cf** under the **ZONES** tab, if it is not already open. Select the **Edit Filter Chain** button.

   <img src="/nginx/images/ns1-edit-filter-chain.png" alt="Screenshot of NS1 GUI: selecting Edit Filter Chain button" width="1024" height="664" class="aligncenter size-full wp-image-63018" />

2. In the **Add Filters** window that opens, select the plus sign (+) on the box labeled **Shed Load** in the **HEALTHCHECKS** section.

   <img src="/nginx/images/ns1-add-filters-shed-load.png" alt="Screenshot of NS1 GUI: selecting Shed Load button on Add Filters page" width="1022" height="701" class="aligncenter size-full wp-image-63017" />

3. The **Shed Load** filter is added as the fourth (lowest) box in the **Active Filters** section. Move it to be third by selecting and dragging it above the **Select First N** box.

4. Select the **Save Filter Chain** button.

5. Back on the ``A`` record's details page, in the **Filter Chain** column select the **Shed Load** box, which expands to display an explanation of how the filter works. Select the label on the white box at the bottom of the explanation and select **Active connections** from the drop-down menu.

   <img src="/nginx/images/ns1-shed-load-active-connections.png" alt="Screenshot of NS1 GUI: selecting Active connections for shed filter" width="1024" height="959" class="aligncenter size-full wp-image-63016" />

6. In the **Ungrouped Answers** section, select the stacked dots icon at the right end of the field for the US-based NGINX Plus instance (10.10.10.1) and select **Edit Answer Metadata**.

   <img src="/nginx/images/ns1-edit-answer-metadata-connections.png" alt="Screenshot of NS1 GUI: selecting Edit Answer Metadata button for shed filter" width="1024" height="204" class="aligncenter size-full wp-image-63015" />

7. In the **Answer Metadata** window that opens, set values for the following metadata. In each case, select the icon in the **FEED** column of the metadata's row, then select or enter the indicated value in the **AVAILABLE** column. (For testing purposes, we're setting very small values for the watermarks so that the threshold is exceeded very quickly.)

   - **Active connections** – **us-nginxgslb-datafeed**
   - **High watermark** – **5**
   - **Low watermark** – **2**

   After setting all three, select the **Ok** button. (The screenshot shows the window just before this action.)

   <img src="/nginx/images/ns1-shed-filter-metadata.png" alt="Screenshot of NS1 GUI: Answer Metadata page for shed filter" width="1024" height="1131" class="aligncenter size-full wp-image-63014" />

<span id="testing-the-threshold"></span>

#### Test the threshold

With the shed filter in place, we're ready to verify that NS1 shifts traffic to the next-nearest NGINX Plus instance when the number of active connections on the nearest instance exceeds the high watermark (upper threshold) of 5. As noted in Step 7 just above, we've set a very small value so we can quickly see the effect when it's exceeded. With the low watermark setting of 2, NS1 will start shifting traffic probabilistically when there are three active connections and definitely when there are five or more connections.

We have written a script that continuously simulates more than four simultaneous connections. We have also configured the backend app to perform a sleep, so that the connections stay open long enough for the agent to report the number of active connections to NS1 before they close.

We run the following commands on a host located in the US.

1. Query the NGINX Plus API for the number of active connections:

   ```shell
   $ curl -X GET "127.0.0.1:8000/api/<VERSION>/connections" -H "accept: application/json" | python3 -m json.tool | grep active

   "active": 1,
   ```

2. Query the NS1 API to learn the number of active connections the NS1 agent has reported to NS1. (For details about this API call, see the [NS1 documentation](https://developer.ibm.com/apis/catalog/ns1--ibm-ns1-connect-api/Introduction). If the page doesn't scroll automatically to the relevant section, search for "Get data feed details".)

   On the command line:

      - ``<NS1_API_KEY>`` and ``<NS1_DATA_SOURCE_ID>`` are the same values we included in the YAML file in Step 4 of [Install the NS1 agent](#agent-install) and used in Step 2 of [Verifying Traffic Redistribution When an Upstream Group Is Down](#verify-when-upstream-down).

      - ``<NS1_FEED_ID>`` is the ID assigned by NS1 to the **us-nginxgslb-datafeed** data feed. It was reported as ``<FEED_ID>`` in the ``id`` field of the output in Step 2 in [Verifying Traffic Redistribution When an Upstream Group Is Down](#verify-when-upstream-down). (It also appears in that field in the following output.)

   The relevant field in the output is ``connections`` in the ``data`` section, and in this example it indicates there is one active connection.

   ```shell
   $ curl -X GET -H 'X-NSONE-Key: <NS1_API_KEY>' https://api.nsone.net/v1/data/feeds/<NS1_DATA_SOURCE_ID>/<NS1_FEED_ID> | python3 -m json.tool

   {
     "config": {
       "label": "us-nginxgslb-datafeed"
     },
     "data": {
       "connections": 1,
       "up": true
     },
     "destinations": [
       {
         "destid": "<ANSWER_ID>",
         "desttype": "answer",
         "record": "<RECORD_ID>"
       }
     ],
     "id": "<FEED_ID>",
     "name": "us-nginxgslb-datafeed",
     "networks": [
       0
     ]
   }
   ```

3. Determine which site NS1 is returning for hosts in the US. Appropriately, it's 10.10.10.1, the IP address of the US-based NGINX Plus instance.

   ```shell
   $ nslookup nginxgslb.cf

   Server:     10.10.100.102
   Address:    10.10.100.102#53

   Non-authoritative answer:
   Name:	nginxgslb.cf
   Address: 10.10.10.1
   ```

4. Create five or more connections to the NGINX Plus instance. We do this by running the script mentioned in the introduction to this section.

5. Repeat Step 1. The NGINX Plus API now reports five active connections.

   ```shell
   $ curl -X GET "127.0.0.1:8000/api/<VERSION>/connections" -H "accept: application/json" | python3 -m json.tool | grep active

   "active": 5,
   ```

6. Repeat Step 2. The NS1 API also reports five active connections.

   ```shell
   $ curl -X GET -H 'X-NSONE-Key: <NS1_API_KEY>' https://api.nsone.net/v1/data/feeds/<NS1_DATA_SOURCE_ID>/<NS1_FEED_ID> | python3 -m json.tool

   {
     "config": {
       "label": "us-nginxgslb-datafeed"
     },
     "data": {
       "connections": 5,
       "up": true
     },
     ...
   }
   ```

7. Wait an hour – because we didn't change the default time-to-live (TTL) of 3600 seconds on the ``A`` record for **nginxgslb.cf** – and repeat Step 3. NS1 returns 10.10.10.2, the IP address of the NGINX Plus instance in Germany, which is the nearest now that the instance in the US has too many active connections.

   ```shell
   $ nslookup nginxgslb.cf

   Server:     10.10.100.102
   Address:    10.10.100.102#53

   Non-authoritative answer:
   Name:	nginxgslb.cf
   Address: 10.10.10.2
   ```

---

## Troubleshooting

### The agent doesn't start

**Symptom**: The agent exits with an error about its configuration file.

**Cause**: The YAML file is invalid, or a value is missing. In the `feeds` list, `feed_name` must be indented like `name`, inside the same list item.

**Fix**: Compare your file with the example in [Install the NS1 agent](#agent-install), and with the [configuration reference](https://github.com/nginxinc/nginx-ns1-gslb/blob/master/configs/README.md).

---

### NS1 still returns the old address

**Symptom**: After a change, `nslookup` still returns the address of the instance that is down or overloaded.

**Cause**: Resolvers cache the `A` record for its time-to-live (TTL), 3600 seconds by default.

**Fix**: Wait for the TTL to pass. For tests, set a shorter TTL on the record.

---

## References

- [The nginx-ns1-gslb project on GitHub](https://github.com/nginxinc/nginx-ns1-gslb)
- [IBM NS1 Connect documentation](https://www.ibm.com/docs/en/ns1-connect)
- [Live activity monitoring]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}})

---

## Revision history

- Version 2 (November 2026): Restructured to the how-to template. Notes that the agent is archived, and fixes the YAML example, the Go version, the start command, and the NS1 links.
- Version 1 (September 2019): Initial version (NGINX Plus Release 19)
