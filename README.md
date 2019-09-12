# python-review-zoo



## sys.argv usage

Consider the following code sample:

```python
def main():
    if len(sys.argv) < 4:
        raise Exception("Cannot work with less than 3 arguments")
    # TODO check if files exists?

    files = Files(
        ancestor=sys.argv[1],
        mine=sys.argv[2],
        yours=sys.argv[3]
    )
    dry_run = "-d" in sys.argv

    merger = Merger(files=files)
    merger.merge(dry_run=dry_run)


if __name__ == "__main__":
    main()
```

In such case, it is strongly suggested to use [argparse](https://docs.python.org/3/library/argparse.html). Unless you are referencing `sys.argv[1]` in a one-off script, you should always use argparse to accept command line arguments.


## usage of `mkstemp`

Consider the following class:

```python
class Temporal(object):
    def __init__(self, files_contents):
        self.locations = {}
        for version, content in files_contents.items():
            fd, file_name = tempfile.mkstemp()
            self.locations[version] = file_name
            with open(fd, 'w') as opened:
                opened.write(content)

    def remove_all(self):
        for _, file_name in self.locations.items():
            os.remove(file_name)
```
From python docs `mkstemp() and mkdtemp() are lower-level functions which require manual cleanup.`

`fd` here is left dangling. See `https://docs.python.org/3/library/os.html#os.close` - file descriptors opened by `tempfile.mkstemp()` need to be closed manually or they leak. If you will cross the limit of open file descriptors on your system (usually 1024 per user), you'll start getting random errors.

Also, you don't provide any prefix or suffix to `mkstemp`. If your program dies unexpectedely, a file will be left in `/tmp` and it will be hard to distinguish it from other files that might be needed for something. Please provide a prefix.

## NotImplementedError

Consider the following class:

```python
class BaseMerger(object):
    def __init__(self, files):
        self._files = files

    def merge(self):
        raise NotImplementedError("Implemented in subclass")
```

In such case, it is strongly suggested to use the [abc metaclass](https://docs.python.org/3/library/abc.html). This way, in case you forget to implement a method in one of the children, you will get the error when the class is being read and not when the object is constructed.

## variable passed directly to %

see here: https://www.python.org/dev/peps/pep-0498/#rationale

## invalid import order

see here: http://isort.readthedocs.io/en/latest/#how-does-isort-work

## catching too much

Consider the following method:

```python
def download_page_with_retries(self, url):
    retries_num = 0 
    while retries_num <= self.retries_limit:
        try:
            proxy = self.get_proxy()
            html_page, http_code = self.download_page(url, proxy)
            if http_code == 200:
                return html_page
        except requests.exceptions.RequestException as e:
            self.notice_proxy_error(proxy)
        retires_num =+ 1
    ...
```
if `proxy = self.get_proxy()` will one day have a new implementation which sometimes raises `requests.exceptions.RequestException`, it will be silently caught and a proxy will get downranked. Better move that line out of the `try` block. Generally try to keep as little as possible in there to avoid catching something accidentally.

## formatting logs

Consider the following line:

```python
logging.info("Spawned thread: {}".format(thread.name))
```
potentially expensive `format()` should only be called in case log level is set in a way that allows the given log statement to be emitted. `logging.info("Spawned thread: %s", thread.name)` uses `%` only if necessary.

## django order_by annotation phenomenon

```pycon
>>> list(SomeModel.objects.values('code').annotate(points=Count('date')).values_list('code', 'points'))
[('some_code', 1), ('some_code', 1), ('some_code', 1)]
```
but:
```pycon
>>> list(SomeModel.objects.order_by().values('code').annotate(points=Count('date')).values_list('code', 'points'))
[('some_code', 3)]
```
(`.values(...).annotate(...)` is a Django idiom for `GROUP BY`, and `Count('date')` counts rows with `NOT NULL` `date` column)

what is happening is inside `SomeModel.Meta`:
```
ordering = ['other', 'date']
```
and Django automatically adds, those to `GROUP BY` to be able to `ORDER BY` those fields

What is even worse - `other` is a `models.ForeignKey(OtherModel)` field and `OtherModel.Meta` has `ordering = ['name']` - so Django automatically `JOIN` tables to acquire `other__name` and is ordering by this field

Therefore:
- clean up ordering by `.order_by()` before `.values(...).annotate(...)` `GROUP BY` pattern
- maybe don't use `Meta.ordering`?

## raising Exception with a message

Consider the following method:

```python
def get_proxy(self):
    with self.lock:
        if len(self.proxies) > 0:
            return random.sample(self.proxies, 1)[0]
        else:
            raise Exception("Lack of working proxies")
```

the only way to catch this is to `except Exception`, compare the string and re-raise if we caught the wrong thing. Please don't force the user of this class to do this. Instead, inherit from `Exception` like this:

```python
class YourModuleException(Exception):
    pass

class NoViableProxies(YourModuleException):
    pass
```
then the user can catch all of your exceptions or just the one and handle it appropriately.

## formatting multiline statements

Consider the following code snippet:

```python
if response_html is None: 
    logging.info(
        "Aborting search for keyword: %s. Cannot download this page",
        job.start_url
    )
```
When the log template line will be extended to accept another parameter, `job.start_url` line has to be deleted and re-added with a `,` after it. This may not be very bad, but it makes it harder for the reviewer to see what the change is about. If the trailing comma was provided, the diff would show the template change and then a line added. Therefore, end the multi-line function calls with a trailing comma. The same **also applies to lists, sets, dicts and so on**, when they are created in multiple lines.


## usage of itertools chain

Consider the following code snippets:

```python

for app_node in itertools.chain(*self._app_nodes.values()):
    app_node.destroy()

```


```python
for app_nodes in self._app_nodes.values():
    for app_node in app_nodes:
        app_node.destroy()
```
2-nested loop is not so bad, itertools.chain and tuple unfolding version of the code is correct (aldough it consumes more memory), but which version would you prefer to support at 3:43 AM when a production system is crashing and burning and you were woken up by a monitoring operator to fix it? I know I would go with the nested loop, it doesn't make me think as much as the first one.

## manual memoization 

```python

class A:
    def __init__(self):
        self._cached_value = None
    @property
    def b(self):
        if self._cached_value is None:
            print('generating')
            self._cached_value = self._helper_method()
        return self._cached_value
```

vs

```python
class A:
    @property
    @functools.lru_cache(1)
    def b(self):
        print('generating')
        return self._helper_method()
```
(actually the helper method could be inlined here, making the code even cleaner)

This is how it works:
```pycon
>>> a = A()
>>> a.b
generating
123
>>> a.b
123
>>> a.b
123
>>> 
```
while it uses slightly more memory than the first version, it makes the code much cleaner.

## how to efficiently check if a loop was entered or not

http://python-notes.curiousefficiency.org/en/latest/python_concepts/break_else.html#but-how-do-i-check-if-my-loop-never-ran-at-all

and

https://github.com/bleachbit/bleachbit/commit/a5cbfbcec7705c35bd96792bb06d70fea125985c

## quite a few well-described anti-patterns

https://docs.quantifiedcode.com/python-anti-patterns/correctness/index.html

## Requests timeout

Remember to set `timeout` in all requests in production code.

http://docs.python-requests.org/en/master/user/quickstart/#timeouts

http://docs.python-requests.org/en/master/user/advanced/#timeouts


## Usage of __subclass__

This language feature is an accident: https://mail.python.org/pipermail/python-list/2003-August/210297.html

It's dangerous, because if you'd like to subclass an interface in an incompatible way but a (possibly indirect (if recursive)) parent uses this in a classmethod to generate something like a registry, but you've added a mandatory function argument, the parent will crash. This is really funny if the parents add the classmethod in a version that is created _after_ you have subclassed and then it crashes your deployment even if both sides use best practices for versioning such as SemVer and appropriate version pinning.

Instead, explicitly use a decorator to register. We usually use https://class-registry.readthedocs.io/en/latest/iterating_over_registries.html
