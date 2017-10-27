I've been experimenting with Airflow quite a bit recently. One thing that becomes clear very quickly is just how little thought has been put into facilitating ease of testing (from an end-user point of view). Not only is it very difficult to orchestrate DAGs for an integration test, there isn't any easy way to fake the context for a unit test.



If you use XCOM as the primary way of passing data between the tasks, I'll show you how to mock the key pieces to make unit tests possible.



First, it helps to have a dummy task abstraction, which is required to instantiate a TaskInsance object:

    {% highlight python %}
    class FakeTask(object):
        def __init__(self,
                     task_id='test_task_id',
                     dag_id='test_dag_id',
                     queue=None,
                     pool=None,
                     run_as_user='test-user',
                     priority_weight_total=1):
            self.task_id = task_id
            self.dag_id = dag_id
            self.queue = queue
            self.pool = pool
            self.run_as_user = run_as_user
            self.priority_weight_total = priority_weight_total
    {% endhighlight %}

Then we can mock TaskInstance's `xcom_pull` method to return the desired data:

    {% highlight python %}
    operator = OperatorUnderTest(...)
    t = FakeTask(task_id="test-task")
    ti = TaskInstance(t, '2017-10-01')
    ti.xcom_pull = MagicMock(return_value="fake xcom data")
    operator.execute({"task": t, "ti": ti})
    {% endhighlight %}
