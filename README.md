# PhotoRoom Infrastructure Engineer Technical Test

First we are going to calculate the upper bound for the number
on instances needed to process the incoming workload.


```python

# loaded from inputs:
# MODEL_LATENCY: { modelName -> latency } dictionary
# REQUESTS: the requests array


# sum on all the requests latency 
total_usage_time = 0
# number of request per model
requests_per_model = dict()
# per model cumulative duration of the requests
cum_time_per_model = dict()
total_requests = 0
start_time = None
end_time = 0

for request in REQUESTS:
	model = request['model']
	latency = MODEL_LATENCY[model]

	total_usage_time += latency
	cum_time_per_model[model] += latency
	
	requests_per_model[model] += 1
	total_requests += 1


	# keep track of the time
	request_time = request['requestTime']
	if start_time is None:
		start_time = request_time
	end_time = max(end_time, request_time + 2*latency)

# the total time span of the requests
total_time = end_time - start_time

# we double the usage as the worst case is that we'll
# need to load the model every time.
num_instances_max = ceil(total_usage_time*2 / total_time)
num_instances_min = ceil(total_usage_time / total_time)
```

We can normalize the cumulative time to calculate the share of processing
each model require:

```python

weights_per_model = dict()

for model, usage in cum_time_per_model:
	weights_per_model[model] = float(usage) / float(total_usage_time)
```


We now have the upper and lower bound of the number of instances we need to
handle the workload, as well as relative usage weights for each model.


```

```