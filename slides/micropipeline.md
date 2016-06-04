## Write your own micro data processing framework in python


# GliaCloud
A company I fund with friends from Python Community

#### Focus on DATA / AI.
#### GliaStudio: an AI Video Producer
Check our website: www.gliacloud.com for more details


## Agenda
- Brief of Data Processing Framework  (5min)
- The Pipeline pattern used in data processing framework and how pipeline works (10min)
- Django-P, a micro data processing framework written in django (10min)



### There are lots of data processing framework
- PySpark
- Data Flow (written by Google)
- MapReduce (written by Google)
- TaskFlow (part of Open Stack)
- Luigi (contributed by Spotify)
- Scikit-Learn


### A very simple abstraction of data processing framework
- Application Layer (such as MapReduce / Hive)
- Pipeline Layer (describe later ...)
- Task Execution Layer (Message Queue, Clustering)


## What is Pipeline?
> ... to connect together complex, time-consuming workflows (including human tasks).

___Let's explain with some examples___


### TaskFlow (OpenStack)
```
class TaskA(task.Task):
    default_provides = 'a'

    def execute(self):
        print("Executing '%s'" % (self.name))
        return 'a'

class TaskB(task.Task):
    def execute(self, a):
        print("Executing '%s'" % (self.name))
        print("Got input '%s'" % (a))

wf = linear_flow.Flow("pass-from-to")
wf.add(TaskA('a'), TaskB('b'))
```

___It controlled that to run TaskB, the TaskA should be done first. And the output of TaskA will become TaskB's input later. TaskFlow also included `Linear`, `Unordered`, and `Graph` flow and the ability to combine them together and create a really complex flow controller___


### Luigi
> Luigi is a Python module that helps you build complex pipelines of batch jobs. It handles dependency resolution, workflow management, visualization etc.

___the same purpose with different approach___


### Luigi
```
class Foo(luigi.Task):
    def run(self):
        pass

    def requires(self):
        for i in range(30 / max_depth):
            current_nodes += 1
            yield Bar(i)

class Bar(luigi.Task):
    num = luigi.IntParameter()

    def run(self):
         pass

    def requires(self):
        if max_total_nodes > current_nodes:
            valor = int(random.uniform(1, 30))
            for i in range(valor / max_depth):
                current_nodes += 1
                yield Bar(current_nodes)
```
___Instead of a separate flow controller. It used a `require` method to define the dependency. While Foo runs, all Bar defined in requires will be check, Any not finish Bar Task will be trigger to run.___


### Luigi
![](https://raw.githubusercontent.com/spotify/luigi/master/doc/user_recs.png)


### (py)Spark

```
@inherit_doc
class Pipeline(Estimator, MLReadable, MLWritable):
    """
    A simple pipeline, which acts as an estimator. A Pipeline consists
    of a sequence of stages, each of which is either an
    `Estimator` or `Transformer`.
    """
```

```
tokenizer = Tokenizer(inputCol="text", outputCol="words")
hashingTF = HashingTF(inputCol=tokenizer.getOutputCol(), outputCol="features")
lr = LogisticRegression(maxIter=10, regParam=0.01)

pipeline = Pipeline(stages=[tokenizer, hashingTF, lr])
```
___In this case, the pipeline used to chain heavy NLP task. The scikit-learn pattern is also similar___


### DataFlow (Google)

```
class Pipeline(object):
  """A pipeline object that manages a DAG of PValues and their PTransforms.

  Conceptually the PValues are the DAG's nodes and the PTransforms computing
  the PValues are the edges.

  All the transforms applied to the pipeline must have distinct full labels.
  If same transform instance needs to be applied then a clone should be created
  with a new label (e.g., transform.clone('new label')).
  """
```
```
p = df.Pipeline(options=pipeline_options)

(p
 | df.io.Read(df.io.TextFileSource(my_options.input))
 | df.FlatMap(lambda x: re.findall(r'[A-Za-z\']+', x))
 | df.Map(lambda x: (x, 1)) | df.combiners.Count.PerKey()
 | df.io.Write(df.io.TextFileSink(my_options.output)))

p.run()
```
___Just like Spark except it use the bash-like way to chain tasks___


## What is pipeline
- Task execution (async, clustering)
- Flow control  (order, dependency, branch)
- Workflow reuse (don't repeat)



## Django-p
#### A django-native micro data processing pipeline inspired by Google Pipeline API
initialed by me, v0.1 now. (means it is still POC now)

https://github.com/lucemia/django-p


### Why called django-p?
Because I used django-q as the under task / operation execution layer.

### django-Q
> A multiprocessing distributed task queue for Django

___I am lazy so I name after django-q___


### What is Google Pipeline API

> The purpose of the Pipeline API is to connect together complex, time-consuming workflows (including human tasks). The goals are flexibility, workflow reuse, and testability. Importantly, no tasks or CPU are consumed while workflows block on external events, meaning many, many workflows can be in flight at the same time with minimal resource usage.

___Sounds good?___


### Why Google Pipeline API
> ...enabling developers to express data dependencies while achieving parallelism.

```
class AddOne(Pipe):
    def run(self, number):
        return number + 1

class AddTwo(Pipe):
    def run(self, number):
        v = yield AddOne(number)
        yield AddOne(v)
```
___Easy to see the dependency, no extra flow controller, and require method___


### Why django?
Because GliaCloud love django!!

django-P use django nice ORM to store the task and the dependency between tasks.


## Design
* Pipeline
* Slot
* Barrier
* Pipe
* Future


### Pipe
```
class Pipe(object):

    def __init__(self, *args, **kwargs):
        self.pk = None
        self.args = args
        self.kwargs = kwargs
        self.class_path = "%s.%s" % (self.__module__, self.__class__.__name__)
        self.output = None

    def run(self, *args, **kwargs):
        raise NotImplementedError()

    def start(self):
        self.save()
        async(evaluate, self.pk)
```
___An abstraction class for Pipeline, define the time-consuming task by overrideing the `run` method, the class_path identify later, where the engine should find the task___


### Pipe
```
class HeavyWork(Pipe):
    def run(self, urls):
        # some heavy task
        pass
```
___Just inherit it and override the run method. The Pipe is the only class user needs to know.___


### Future
```
class Future(object):

    def __init__(self, pipe):
        self._after_all_pipelines = {}
        self.output = pipe.output
```
___Internal class, which hold the pipeline return results, it has two state. Waiting means the result is not ready yet, Done means the result is done finish, the `_after_all_pipelines` recorded all dependency. So the django-p knows how many pipelines need to be done before current one can fire to run.___


### Pipeline
```
class Pipeline(models.Model):
    class_path = models.CharField(max_length=255)
    root_pipeline = models.ForeignKey(
        "Pipeline", null=True, blank=True, related_name="descendants")
    parent_pipeline = models.ForeignKey(
        "Pipeline", null=True, blank=True, related_name="children")

    output = models.ForeignKey("Slot", null=True)
    params = JSONField(default={}) # args, and kwargs, may also reference to Slot

    status = models.IntegerField(choices=STATUS, default=STATUS.WAITING)
```
___Store pipeline config to db, the parameters store all arguments of the pipeline, it can store values or a reference to another pipeline's output (Slot)___


### Slot:
```
class Slot(models.Model):
    filler = models.ForeignKey(Pipeline, null=True)

    value = JSONField(default=None)
    status = models.IntegerField(choices=STATUS, default=STATUS.WAITING)

    filled = models.DateTimeField(auto_now=True)
```
___Each pipeline has a Slot, which store pipeline execution results___


### Barrier
```
class Barrier(models.Model):
    target = models.OneToOneField(Pipeline)

    blocking_slots = models.ManyToManyField(Slot)
    triggered = models.DateTimeField(null=True, auto_now=True)
    status = models.IntegerField(choices=STATUS, default=STATUS.WAITING)
```
___Each pipeline also has a barrier. Barrier is a special class used to prevent pipeline run before all dependencies are satisfied already.___



## Define a Pipe

```
class AddOne(Pipe):
    def run(self, number):
        return number + 1

class AddTwo(Pipe):
    def run(self, number): # 1. a generator
        # 2. yield a Pipe and get the result from control loop
        v = yield AddOne(number)                      --- Pipe A
        # 3. padd the result to another pipe
        yield AddOne(v)                               --- Pipe B
```

The simple code tells us a lot:
1. It imply Pipe B is depend on Pipe A, so PipeA should run first
2. It imply AddTwo return the same value as PipeB
3. It imply Pipe B is a heavy task and Pipe A is a light task


### Main control loop
```
pipeline_iter = pipeline.run(*args, **kwargs)

while True:
    try:
        yielded = pipeline_iter.send(next_value)
    except StopIteration:
        breaking
    except Exception, e:
        raise

    assert isinstance(yielded, Pipe)
    next_value = Future(yielded)
    child_pipeline = yielded

    dependent_slots = set()
    for arg in child_pipeline.args:
        if isinstance(arg, Slot):
             dependent_slots.add(arg)
    for key, arg in child_pipeline.kwargs.iteritems():
       if isinstance(arg, Slot):
            dependent_slots.add(arg)
    for other_future in future._after_all_pipelines:
        slot = other_future.output
        dependent_slots.add(slot)

    barrier = Barrier.objects.create(
        target_id=child_pipeline.pk,
    )
    barrier.blocking_slots.add(*dependent_slots)
    notify_barrier(barrier)
```
___I promissed, it is the most complicate code in this talk___


### In Short
___The control loop use python's yield feature to do all the magic behind, A pipeline may execute serveral child pipelines by `yield` them.  The control loop processed these pipeline one by one and transform them to future with dependency informations inside. The future send back to current pipeline with `.send` and the pipeline pass these future to other child pipelines.___


### Another Example
```
class WordCountUrl(pipeline.Pipeline):
  def run(self, url):
    r = urlfetch.fetch(url)
    return len(r.data.split())

class Sum(pipeline.Pipeline):
  def run(self, *values):
    return sum(values)

class MySearchEnginePipeline(pipeline.Pipeline):
  def run(self, *urls):
    results = []
    for u in urls:
      results.append( (yield WordCountUrl(u)) )
    yield Sum(*results) # Barrier waits
```
___task can easily spread over cluster___


### With django-P
1. dependency resolution automatically.
2. workflow management
2. Execute tasks async on clusting (by django-q)



## Advanced Flow Control


## InOrder
```
class LogWaitLogInOrder(pipeline.Pipeline):

  def run(self, message1, message2, delay):
    with pipeline.InOrder():
      yield LogMessage(message1)
      yield Delay(seconds=delay)
      yield LogMessage(message2)

    yield LogMessage('This would happen immediately on run')
```


## After
```
class LogWaitLogAfter(pipeline.Pipeline):

  def run(self, message1, message2, delay):
    first = yield LogMessage(message1)
    with pipeline.After(first):
      delay = yield Delay(seconds=delay)
      with pipeline.After(delay)
        yield LogMessage(message2)

      yield LogMessage('This would happen after the first message')

    yield LogMessage('This would happen immediately on run')
```


## Future Work
* idempotent Pipeline
* Life Cycle Control
* fully asynchronous and even call out to human operators to decide how the pipeline should proceed.

___please contribute!___
