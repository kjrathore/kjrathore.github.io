---
layout: single
title:  "Data retrival from USACE swagger api"
header:
  teaser: "unsplash-gallery-image-2-th.jpg"
categories: 
  - Jekyll
tags:
  - edge case
---

How to use swagger api to collect data?
======


Collect data from USACE Swagger page:
======
Source site: https://cwms-data.usace.army.mil/cwms-data/swagger-ui.html

Steps to use data swagger for data collection on various dam sites:
  1. Visit : https://cwms-data.usace.army.mil/cwms-data/swagger-ui.html
  2. Check the office under which the target Dam/waterbody lies: https://nid.sec.usace.army.mil/#/
  3. On Swagger page, check the list of offices, if your Dam office exists in the list.
  4. Since the office name is used in multiple apis.
  In my case > 
  ```
  {
          "name": "NWP",
          "long-name": "Portland District",
          "type": "DIS",
          "reports-to": "NWDP"
        },
  ```
  5. Using "NWDP" as office-id,        use GET: /cwms-data/location/group
  Extract relevant info from output:

  ```
  {
      "office-id": "NWDP",
      "id": "DET",
      "location-category": {
        "office-id": "NWDP",
        "id": "RDL_Project_Stations",
        "description": "Associate stations with particular Projects"
      },
      "description": "Detroit Corps Project",
      "shared-loc-alias-id": "Willamette:North_Santiam",
      "assigned-locations": []
    },
    {
      "office-id": "NWDP",
      "id": "DET_BCL",
      "location-category": {
        "office-id": "NWDP",
        "id": "RDL_Project_Stations",
        "description": "Associate stations with particular Projects"
      },
      "description": "Detroit and Big Cliff Corps Projects",
      "assigned-locations": []
    },
  ```

  6. Cross check the data using 
  ```
  GET : /cwms-data/locations/{location-id}
  ```
  where 'location-id' = 'id' from Step-5 output. A response will appear with Code 200.

  7. using office-id. check the target timeseries ids from : 
  ```
   GET: /cwms-data/timeseries/group
  ```
  8. On swagger page: check/download the list of parameters and their codes.
  

