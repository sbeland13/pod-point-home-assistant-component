
# Pod Point HACS Integration Feedback
### by Stephane Beland

## Summary
Overall, I do like the fact that the PodPoint API interface was put into its own podpointclient library. I think the way the api integration is set up will work fine for this use case where it is distributed across 1000's of installs, however there are some minor improvements that can be made:

The body of the API response from PodPoint contains a lot of data that we may not be utilized in the HA component. In [`client.py  -> async_get_pods(self)`](https://github.com/mattrayner/podpointclient/blob/c46b0a65436ef16e7b0bd29b71fdf2923f567b58/podpointclient/client.py#L46) we have:
```python
includes = ["statuses", "price", "model", "unit_connectors", "charge_schedules"]
params = {"perpage": "all", "include": ",".join(includes)}
```
One small improvement would be to enable pagination for this similar to the [`async_get_charges`](https://github.com/mattrayner/podpointclient/blob/c46b0a65436ef16e7b0bd29b71fdf2923f567b58/podpointclient/client.py#L108) method and have the `includes` values set as a keyword argument with a default value which can be overridden if we require less or more fields depending on the application using the client.
```python
DEFAULT_INCLUDES = ["statuses", "price", "model", "unit_connectors", "charge_schedules"]

async  def  async_get_pods(self, perpage=5, page=1, includes=DEFAULT_INCLUDES) ->  List[Pod]:
```

Though unlikely that most users would own more than a few of these chargers where this would be a problem, one area where this could be useful is where we call `async_get_pods` from [`config_flow.py -> _test_credentials`](https://github.com/sbeland13/pod-point-home-assistant-component/blob/d4bc3b0181810bd0df2dfc6426f5b03038839f60/custom_components/pod_point/config_flow.py#L97), where we don't need to return the whole extended payload just to test connectivity.

On the topic of results pagination, one thing that is missing from the code is for example in [`client.async_get_charges`](https://github.com/mattrayner/podpointclient/blob/c46b0a65436ef16e7b0bd29b71fdf2923f567b58/podpointclient/client.py#L108), we use the pagination feature of the api by setting the default `perpage` param to 5 as well as setting the `page` param to 1, but there is no pagination handler to handle fetching of subsequent pages. This would make it impossible to fetch beyond 5 charge sessions if the kwarg is not overridden with the value `"all"`. I propose something like the following in `podpointclient/client.py : PodPointClient`:

```python
async def paginated_get(self, url: str, params: Dict[str, Any], headers: Dict[str, Any]): -> List[Dict[str, Any]]
    response_results = []
    page = int(params.get(page, "1"))
    new_results = True

    response_results.extend(json.get("results", {}))  # would need to test for the right "results" json key here for diffent endpoints
    next_url = json.get("next_url")
    while new_results:
        response = await self.api_wrapper.get(url=url, params=params, headers=headers)
        json = response.json()
        new_results = json.get("results", {})
        response_results.extend(new_results)
        page += 1
        params["page"] = str(page)
    return response_results
```

And then we would call this method directly in [`client.async_get_charges`](https://github.com/mattrayner/podpointclient/blob/c46b0a65436ef16e7b0bd29b71fdf2923f567b58/podpointclient/client.py#L120):
```diff
-  response = await self.api_wrapper.get(url=url, params=params, headers=headers)

-  json = await response.json()
+  json = await self.api_wrapper.paginated_get(url=url, params=params, headers=headers)

   factory = ChargeFactory()
-  charges = factory.build_charges(charge_response=json)
+  charges = []
+  for res in json:
+      charges.extend(factory.build_charges(charge_response=res))
```


## Nit/Minor code cleanliness
Overall did not find many issues.
- There are a few instances in unit tests where `==` is used to assert a `None` value instead of `is`, example [here](https://github.com/sbeland13/pod-point-home-assistant-component/blob/d4bc3b0181810bd0df2dfc6426f5b03038839f60/tests/test_coordinator.py#L61).
- Some minor code smell were private class members are accessed externally, for example [here](https://github.com/sbeland13/pod-point-home-assistant-component/blob/d4bc3b0181810bd0df2dfc6426f5b03038839f60/tests/test_sensor.py#L115)
