---
title: "Cloud Sizing"
date: 2022-09-29T19:33:55+02:00
draft: false
---
Some days ago, based on a question from a customer and as a way to "dust off" my python skills, i started writing a quick script/tool that fetches the assigned public ips from the major cloud providers (AWS, GCP, Azure) and converts it to other formats ready to be pushed to network infrastructure (Firewalls, ACLs, LBs, Routers, BGP injection, etc).

While doing this, and inspired from a "100M AWS IPv4 addressess" number, researched by Andre Toonk that i read in another blog, i got inspired to check the current status of the IPv4 usage on the cloud providers, and compare their size based on the amount of addresses assigned to their services.

First we start with actually getting the data in. Where comes the data from? GCP and AWS very kindly provide a quick way of fetching a .json file with a simple GET that contains all their assigned IP ranges. Google does it for both "GOOG" (Core services) and Google Cloud (GCP). The GCP data is also divided in regions but does not contain more info. For the purpose of this simple script, this is more than enough.

GOOGLE: https://www.gstatic.com/ipranges/goog.json

GCP: https://www.gstatic.com/ipranges/cloud.json

AWS: https://ip-ranges.amazonaws.com/ip-ranges.json

Azure was a little more tricky, but we´ll talk about that later, lets get working on our python. In order to fetch this data programmatically and start playing with it, i created a basic GET function using the requests module that does the HTTP get and returns the json response:

```python
def basicGET_JSON(url: str) -> dict:
    retries = 3
    for n in range(retries):
        try:
            response = requests.get(url)
            response.raise_for_status()
            break
        except HTTPError as exc:
            code = exc.response.status_code
            if code in [500, 502, 503, 504]:
                # retry after n seconds
                time.sleep(n)
                continue
            raise
    return response.json()
```

So, reusing this function, I define another one to fetch GCP and one to fetch AWS. Two functions are not actually needed, but i want some modularity as the end goal is to apply some filtering and extra functionallity for the final tool. For the sake of brevity, lets just look the GCP one, as the AWS and GOOG looks almost exactly the same *(ignore the "f" filters for now, as thats part of a later functionality that im planning)*:

```python
def fetchGCP(f=["all", "ipv4"]):
    GCP_URL = "https://www.gstatic.com/ipranges/cloud.json"
    GCP_JSON = basicGET_JSON(GCP_URL)
    if "ipv4" in f:
        # Going through the json and extracting all ipv4 prefixes into a list
        GCP_PREFIXES = [_.get("ipv4Prefix") for _ in GCP_JSON["prefixes"]]
        # Removing "None" results created by the ipv4-only query
        GCP_PREFIXES = [_ for _ in GCP_PREFIXES if _ != None]
    elif "ipv6" in f:
        # Going through the json and extracting all ipv6 prefixes into a list
        GCP_PREFIXES = [_.get("ipv6Prefix") for _ in GCP_JSON["prefixes"]]
        # Removing "None" results created by the ipv6-only query
        GCP_PREFIXES = [_ for _ in GCP_PREFIXES if _ != None]
    return GCP_PREFIXES
```

The Amazon one is actually easier as is divided on ipv4 and ipv6 prefixes, therefore there is no need to do any "cleaning" after filtering.

An example of the list that we get from GCP function is:

```python
>>> pprint.pprint(test)
['34.80.0.0/15',
 '34.137.0.0/16',
 '35.185.128.0/19',
 '35.185.160.0/20',
 '35.187.144.0/20',
 '35.189.160.0/19',
 '35.194.128.0/17',
 '35.201.128.0/17',
 '35.206.192.0/18',
 '35.220.32.0/21',
 '35.221.128.0/17',
 '35.229.128.0/17',
 '35.234.0.0/18',
...
```

Now into Azure. For some unknown reason, Microsoft does not provide a direct link into their .json file, but they provide a [website](https://www.microsoft.com/en-us/download/confirmation.aspx?id=56519) with a button that contains the actual file, but I couldnt use the direct link as the file name is updated everytime the file changes. So, for Azure, i was forced to do first a GET towards the first website, parse the response looking for the .json link, and then downloading the json.

After i have the json, i apply the same "filtering" as i did for Google or AWS:

```python
def fetchAzure(f=["all", "ipv4"]):
    # Azure download url
    AZURE_URL = "https://www.microsoft.com/en-us/download/confirmation.aspx?id=56519"
    # Get the http code
    AZURE_HTTP = requests.get(AZURE_URL)
    # Parse the code with regex to get the actual .json link
    AZURE_JSON_URL = re.search(
        r"https:\/\/download\.\S*\.json", AZURE_HTTP.text
    ).group()
    # Now use the json getter on the .json link
    AZURE_JSON = basicGET_JSON(AZURE_JSON_URL)
    # The azure file has more details and is a little bit more complex, so there is a need to do more housekeeping.
    TEMP_PREFIXES, AZURE_PREFIXES = [], []
    # Loop throguh the values
    for _ in AZURE_JSON["values"]:
    # Make a list of the values
        TEMP_PREFIXES.extend([x for x in _["properties"]["addressPrefixes"]])
    for prefix in TEMP_PREFIXES:
    # There is a of subnets, ipv4/v6, some without v4 and some without v6, so i need to check each string and verify if its v4 or v6
    # Making it a ip_network instance
        address = ipaddress.ip_network(prefix)
    # Checking if its a valid v4
        if "ipv4" in f and isinstance(address, ipaddress.IPv4Network):
            AZURE_PREFIXES.append(prefix)
    # Checking if its a valid v6
        elif "ipv6" in f and isinstance(address, ipaddress.IPv6Network):
            AZURE_PREFIXES.append(prefix)
    return AZURE_PREFIXES
```

Now, with the data, the rest is just math. Lets define some counters and fetch the data:

```python
aws_count, gcp_count, goog_count, azure_count = 0, 0, 0, 0

f = ["ipv4]
gcp_list = fetchGCP(f)
goog_list = fetchGOOG(f)
aws_list = fetchAWS(f)
azure_list = fetchAzure(f)
```

So now, the subnets are on lists. But we want to know the total number of ip addresses on each subnet, so lets use the ipaddress python module, specifically the num_address method for the IPv(4/6)Network classes.

From the [documentation](https://docs.python.org/3/library/ipaddress.html)

```
ipaddress provides the capabilities to create, manipulate and operate on IPv4 and IPv6 addresses and networks."

Network objects

All attributes implemented by address objects are implemented by network objects as well. In addition, network objects implement additional attributes. All of these are common between IPv4Network and IPv6Network, so to avoid duplication they are only documented for IPv4Network. Network objects are hashable, so they can be used as keys in dictionaries.

class ipaddress.IPv4Network(address, strict=True)

    Construct an IPv4 network definition. address can be one of the following:

num_addresses

    The total number of addresses in the network.
```

With this information, lets first define an ip_hosts function that transforms our v4 subnet string into a IPv4Network object:

```python
def ip_hosts(_):
    return ipaddress.IPv4Network(_)
```

Now we can loop through all the list, and update the counters with sum of all the addresses

```python
for _ in gcp_list:
    gcp_count += ip_hosts(_).num_addresses

for _ in goog_list:
    goog_count += ip_hosts(_).num_addresses

for _ in aws_list:
    aws_count += ip_hosts(_).num_addresses

for _ in azure_list:
    azure_count += ip_hosts(_).num_addresses
```

Now, with the counters, is just a matter of printing or doing some math with them:

```python
    print(f"Number of GCP addresses: {gcp_count:,}")
    print(f"Numer of GOOG addresses: {goog_count:,}")
    print(f"Total GOOG (Google backbone/services + GCP): {gcp_count + goog_count:,}")
    print(f"Total AWS addresses: {aws_count:,}")
    print(f"Total Azure addresses: {azure_count:,}")
    print(f"\n Total cloud: {gcp_count + goog_count + aws_count + azure_count:,}")
```

The final result is:

```
Number of GCP addresses: 11,402,496
Numer of GOOG addresses: 19,065,856
Total GOOG (Google backbone/services + GCP): 30,468,352
Total AWS addresses: 136,870,388
Total Azure addresses: 60,135,143

 Total cloud: 227,473,883
 ```

This means, that based on IPv4 usage the sizing would be: AWS>Azure>GOOG. Which is a surprise for me, as i neverimagined that Azure would be "bigger" than Google core + GCP combined! Maybe Google is more heavy on IPv6 ?
Using the same script, and modifying the filtering, we get:

```
Number of GCP addresses: 716,893,011,031,475,100,600,762,368
Numer of GOOG addresses: 1,902,094,870,361,986,792,382,504,370,176
Total GOOG (Google backbone/services + GCP): 1,902,811,763,373,018,267,483,105,132,544
Total AWS addresses: 1,217,286,314,170,012,654,257,871,781,900
Total Azure addresses: 3,156,118,579,024,291,454,408,966,192

Total cloud: 3,123,254,196,122,055,213,195,385,880,636
```

Which then would be Google > AWS > Azure for ipv6!



A quick search of the total public ipv4 addresses gives the number "3,706,452,992". So, with some quick math, we could say that:

Total GOOG (Google backbone/services + GCP): 30,468,352 -> 0.8%
Total AWS addresses: 136,869,360 -> 3.7%
Total Azure addresses: 60,135,143 -> 1.7%

Total cloud: 227,472,855 -> 6.1%

Now, the next step is to use this data to generate configuration files for Firewalls, LBs, Routers, etc and probably push this data directly to the devices. Also, i want to apply some filtering (based on regions for example) and some extra functionality.
After i finished this quick script i also saw that it would be much better if i return some dictionaries with both ipv4 and ipv6 instead of using a filter on the fuction (as the whole data is fetched on the same GET anyways), that would improve the speed for greater datasets and give me more flexibility. But thats work for another day.