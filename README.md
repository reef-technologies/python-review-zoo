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

Also, you don't provide any prefix or suffix to mkstemp. If your program dies unexpectedely, a file will be left in `/tmp` and it will be hard to distinguish it from other files that might be needed for something. Please provide a prefix.

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
