Variables
=========

Predefined
----------
Variables from inventory, `variables` block or command line `-e` argument

Computed
--------

| Registered in steps variables via `register: {var_name: var_value}`.
| `var_value` can be a variable itself like: `register: {email: '{{ user }}@test.com'}`

Inherited
---------
Inherited from `run` steps. Every run include adds its variables to test scope.

Built-in
--------
| 1. `OUTPUT` - operation's output. Can be used for new variables registration::

    - http:
        post: 
            url: 'http://test.com'
            body: {'id': '{{ id }}', 'action': 'fee'}
        register: {reply: '{{ OUTPUT.id }}'}

| 2. `ITEM` - item of a list. Used in `any` and `all` checks and `foreach` loops ::

    variables:
        list: [{n: 1, k: 'a'}, {n: 2, k: 'a'}, {n: 3, k: 'a'}]
    steps:
        - check:
            all:
                of: '{{ list }}'
                equals: {the: '{{ ITEM.k }}', is: 'a'}

| 3. `NOW_TS` - return timestamp::

    steps:
      - echo: {from: '{{ NOW_TS }}', register: {now: '{{ OUTPUT }}'}}

| 4. `NOW_DT` - return current date in `yyyy-mm-ddTHH:MM:SS0+0000`
| 5. `RANDOM_STR` - return random string in uuid format
| 6. `RANDOM_INT` - return random int [-2147483648, 2147483648]
| 7. `TEST_NAME` - name of the current test.

Environment variables
---------------------

| There is a full support for environment variables in inventory files and partly support in steps itself.
| In steps you can just access them ::

    steps:
        - check: {equals: {the: '{{ FOO }}', is: '1'}}

| If you run `export FOO=1` before - this step will pass.
| There is a limitation (only for steps) - variables should be accessed directly.
| It means, this not work ::

    variables:
        foo: '{{ FOO }}'
    steps:
        - check: {equals: {the: '{{ foo }}', is: '1'}}

| Because here there are 2 steps:

1. replace foo with `{{ FOO }}`
2. replace `{{ FOO }}` with value from environment.

| However, there is no such limitation in inventory.

inventory.yml ::

    example_host: http://example.com
    database_conf:
            host: '{{ DB_HOST }}'
            dbname: '{{ DB_NAME }}'
            user: '{{ DB_USER }}'
            password: '{{ DB_PASSWORD }}'

test.yml ::

    postgres:
    request:
        conf: '{{ database_conf }}'
        query: 'select count(*) from test'

Variables override priority
===========================

Variables from command line
---------------------------
Variables, passed from command line override inventory variables.
`inventory.yaml`::

    foo=bar

in this case `catcher -i inventory.yaml test -e foo=baz` foo variable
will be `baz`.

Variables in test scripts
-------------------------
Variables, set in test scripts, override inventory variables and variables,
passed from command line.
`inventory.yaml`::

    foo=bar

`test.yaml`::

    variables:
        foo=bax
    steps:
        ...

in this case `catcher -i inventory.yaml test.yaml -e foo=baz` foo variable
will be `bax`.

Variables from run includes
---------------------------
Variables, computed via `run` includes override variables declared before.

`compute_fee.yaml`::

    ---
    variables:
      deposit: 50
    steps:
        - echo: {from: '{{ RANDOM_STR }}', register: {uuid: '{{ OUTPUT }}'}}
        # ... do something else

`main_test.yaml`::

    ---
    include:
        file: compute_fee.yaml
        as: compute_fee
    variables:
        deposit: 100
    steps:
        - echo: {from: 'test_user', register: {uuid: '{{ OUTPUT }}'}}
        - check: {equals: {the: '{{ deposit }}', is: 100}}  # deposit is 100, as we set up in variables
        - check: {equals: {the: '{{ uuid }}', is: 'test_user'}}  # uuid is the same we registered several steps above
        - run: compute_fee
        - check: {equals: {the: '{{ deposit }}', is: 50}}  # deposit is 50, computed from compute_fee run
        - check: {equals: {the: '{{ uuid }}', is_not: 'test_user'}}  # uuid is random, got from compute_fee run

Environment variables
---------------------
| All other variables override environmental variables from steps.
| export FOO=bar
test.yml::

    variables:
            FOO: baz
    steps:
        - check: {equals: {the: '{{ FOO }}', is: 'baz'}}

I recommend to use lowercase for your variables and uppercase for environmental.