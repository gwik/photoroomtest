# PhotoRoom Infrastructure Engineer Technical Test

First we are going to calculate the upper bound for the number on instances
needed to process the incoming workload.

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

We now have the upper and lower bound of the number of instances we need to
handle the workload, as well as relative usage weights for each model.

We can normalize the cumulative time to calculate the share of processing each
model require:

```python

weights_per_model = dict()

for model, usage in cum_time_per_model:
	weights_per_model[model] = float(usage) / float(total_usage_time)
```

Let's add a parameter `F` in the range (0.0..1.0], which is a factor that
controls how we want to favor latency over the number of instance:

  - 0.0 -> max latency, minimum number of instances
  - 1.0 -> min latency, maximum number of instances

```python
	number_of_instances = ceil(num_instances_min + F * (num_instances_max - num_instances_min))
```

We can now use a ring of the instances and the weights we have calculated to
load balance the requests:

```python
	N = 10 # number of instances that have been scaled.

	# extracted from the request payload
	model = "modelA"
	weight = model_weights[model]

	# start position in the ring
	start = hash(model) % N

	# size of the ring considering the weight of the model
	length = ceil(weight * N)

	# the final instance number:
	instance = (start + random.uniform(0, length - 1)) % N
```

When the number of instances is small the algorithm could have better
distribution by upscaling the number of instances (vnodes).

The algorithm partition a ring of instances and use the weight to select the
sub-space of instances to route to for this particular model.

It combines ring hashing and weights to effectively route a given model to the
same subset of instances, which size is relative to the model weight. It
maximize the probability that the instances won't have to reload the model.

Instead of modulo hashing we can use the jump hash algorithm or similar to
minimize the change in partitioning when the number of instances grows or
shrinks (answers to the optional assignment).

Autoscaling can be achieved by running this over different time slices and
build a predictive scheduling of the instances.

However the autoscaling could be made in real-time:

The min and max instances we have calculated can be used to set the
autoscaller min and max number of instances (with some added margin).

We can create a custom metric of the GPU usage that we can obtain with tools
like NVML library or nvidia-smi. And send the sample to, e.g, a stackdriver
metric. Then, configure the autoscaler to use that metric as target.


## Addendum

DISCLAIMER: This part was added after the first submission.

The proposed algorithm above has a flaw since it fails to properly distribute
the models around the ring/space according to the respective weight:

```python
start = hash(model) % N
```

The above improperly places the start position at relatively equal intervals
which would make the models ranges to overlap or make a part of the ring
unused.

The following "shares" algorithm would correctly assign non-overlapping slices
of the total space with respect to the relative weight of each model.

The idea is to have a constant number of shares that represent the total space
(or ring). Each instance would be assigned an equal range of the shares. Each
model would be assigned a slice of shares and offset relative to its weight.


```txt

<------------------- total shares / space --------------------->
[   model A ][  model B  ][         model C                    ]
[  instance 0  ][  instance 1  ][  instance 2  ][  instance 3  ]

```

The algorithm will compute a model_name -> slice table that allow to lookup
the slice configuration for a model. The table is built from the normalized
relative weights calculated previously (`weights_per_model`).

When a request comes in, the load balancer will select the target instance by
looking up the slice of the model and selecting a random (or round robin) share
within the slice.

The share is then converted into the instance number:

```txt

                                     selected share
                                       ↓
[   model A ][  model B  ][         model C                    ]
[  instance 0  ][  instance 1  ][  instance 2  ][  instance 3  ]
                                       ↑
                                     selected instance

```

Sample code for the algorithm:

```python

Slice = namedtuple('Slice', ['share_count', 'start_offset'])

def build_model_shares_lookup_table(model_weights, num_shares):
    """ Builds the lookup table for slices.

    `model_weights` is a dictionary model_name -> relative weight, the
    sum of weights must be 1.0.

    `num_shares` is a constant that represent the total space to be divided
    among the models.

    Builds a dictionary model_name -> Slice. Each models
    will be assigned a slice of the available shares relative to the normalized weights.
    """
    offset = 0
    models_lookup_table = dict()
    for model_name, weight in model_weights.items():
        share_count = math.floor(weight * num_shares)
        models_lookup_table[model_name] = Slice(share_count, offset)
        offset += share_count

    return models_lookup_table

def select_instance(model_shares_lookup_table, num_shares, num_instances, model_name):
    """ Selects an instance for a request.

    `model_shares_lookup_table` is the table built with `build_model_shares_lookup_table`

    `num_shares` is total number of shares (as provided when building the lookup table)

    `num_instances` is the total number of instances currently active.

    `model_name` is the model name extract from the request payload.

    Returns the select instance number in [0..num_instances)
    """
    config = model_shares_lookup_table[model_name]
    # select a random point in the ring of shares [start, start + share_count)
    share = math.floor(config.start_offset + random.randint(0, config.share_count - 1))
    return math.floor(share / num_shares * num_instances)

```
