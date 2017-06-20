#Gdelt 接口设计

* GET /search/region/:dateFrom/:dateTo/:regionCode/:level/:eventRootCode/:eventBaseCode/:eventCode

```json
{
    "response": {
        "level": "country",
        "result": [
            {
                "regionCode": "CHN",
                "location": {
                        "lat": 1,
                        "lon": 2
                },
                "record": {
                    "active": 123,
                    "negative": 312,
                    "event": 222
                 },
                "tone": 3.4,
                "impact": 12345,
                "stability": 4.3
            }
         .......
         ]
    }
}
```

* GET /search/actor/:dateFrom/:dateTo/:actorName/:level/:eventRootCode/:eventBaseCode/:eventCode

```json
{
    "response": {
        "level": "country",
        "result": [
            {
                "regionCode": "GOV",
                "location": {
                        "lat": 1,
                        "lon": 2
                },
                "record": {
                    "active": 123,
                    "negative": 312,
                    "event": 222
                 },
                "tone": 3.4,
                "impact": 12345
            }
			.......
         ]
    }
}
```

* GET /trend/region/:from/:to/:regionCode/:level/:eventRootCode/:eventBaseCode/:eventCode

```json
{
    "response": {
        "level": "country",
        "result":  {
            "record": {
                "active": [12345, 23456, 34567],
                "negative": [12345, 23456, 34567],
                "event": [12345, 23456, 34567]
            },
           
            "tone": [1.2, 2.3, 3.4],
            "impact": [456789, 567890, 987654],
            "stability": [3.4, 4.3, 5.4]
        }
    }
}
```

* GET /trend/actor/:from/:to/:actorName/:level/:eventRootCode/:eventBaseCode/:eventCode

```json
{
    "response": {
        "level": "country",
        "result":  {
            "record": {
                "active": [12345, 23456, 34567],
                "negative": [12345, 23456, 34567],
                "event": [12345, 23456, 34567]
            },
            "tone": [1.2, 2.3, 3.4],
            "impact": [456789, 567890, 987654]
        }
    }
}
```


* GET /relation/actor/:from/:to/:actorName (**pending**)

```json
{
    "response": {
        "result": [
        	{
        		"actorName": "USA",
        		"location": {
        			"lat": 1,
        			"lon": 2
        		}
        		"relation":[1, 3]
        	}
         .....
        ]
    }
}
```


* GET /list/:dateFrom/:dateTo/:actor1Name/:actor2Name/:eventRootCode/:eventBaseCode/:eventCode/:page

```json
{
    "response": {
        "result": [
            {
                "Actor1Code": "aaa",
                "Actor1Name": "BBB",
                ...,
                "SourceURL": "www.aaa.com"
            },
            {
                "Acotr1Code": "ccc",
                "Acotr1Name": "DDD",
                ...,
                "SourceURL": "www.ccc.com"
            }
        ]
    }
}
```：