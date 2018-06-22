.. title: Somewhat safer unpickler
.. slug: somewhat-safer-unpicker
.. date: 2018-06-22 10:58:45 UTC
.. tags: python,serialization,python2
.. category: programming
.. type: text

This class inherits all the implementation from the builtin `pickle.Unpickler` but modifies the `self.dispatch` dictionary used to unpack the serialized structures.

This is not checked to be safe, but in case you are using pickled files in production and you are searching for some safer way to load them to convert the data to a different format, this class can be handy.

The same idea works for Python 3, subclassing `pickle._Unpickler`.

<script src="https://gist.github.com/naufraghi/5cabdcb36f20da621956d8412afd1767.js"></script>
