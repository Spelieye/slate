# Metric Description 

```shell 
curl \
  https://geni1.geli.net/api/v1/metrics \
  -H 'Authorization: Token token=\"GENITOKEN"' \
```

> The above command returns JSON structured like this:

```json 
{
    "@dto": "MetricsInfo",
    "metrics": [
        {
            "id": 1000,
            "name": "TRUE_POWER_TOTAL",
            "unit": "WATTS"
        },
        {
            "id": 1010,
            "name": "TRUE_POWER_A",
            "unit": "WATTS"
        },
        {
            "id": 1020,
            "name": "TRUE_POWER_B",
            "unit": "WATTS"
        },
        {
            "id": 1030,
            "name": "TRUE_POWER_C",
            "unit": "WATTS"
        },
        {
            "id": 1100,
            "name": "REACTIVE_POWER_TOTAL",
            "unit": "VAR"
        },
        ...more metrics...
    ]
}
```

Returns all metric IDs and their associated name 

### HTTP Request 

`GET https://geni1.geli.net/api/v1/metrics`

### URL Parameters

No URL Parameters  

### Authorization 

role: USER
