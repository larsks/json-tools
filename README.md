# JSON Utilities

This is a small set of utilities for interacting with JSON on the
command line.  I find it particularly useful for working with
JSON-enabled REST APIs -- such as those used by OpenStack -- on the
command line.

## Requirements

These tools require the Python [dpath][] module.

[dpath]: https://github.com/akesterson/dpath-python

## jsong

`jsong` allows you to generate JSON on the command line by specifying
paths to indicate a JSON hierarchy.  For example, if you run:

    $ jsong auth/passwordCredentials/username=admin \
      auth/passwordCredentials/password=secret

You get:

    {
        "auth": {
            "passwordCredentials": {
                "username": "admin", 
                "password": "secret"
            }
        }
    }

You can also generate data with lists by initializing a path to the
special value `[]`.  For example:

    $ jsong service/type=example \
      service/endpoints='[]' \
      service/endpoints/0='http://foo/' \
      service/endpoints/1='http://bar'

Gets you:

    {
        "service": {
            "endpoints": [
                "http://foo/", 
                "http://bar"
            ], 
            "type": "example"
        }
    }

Note that due to limitations of the underlying `dpath` library, lists
can only be leaf elements (so you can't set
`service/endpoints/0/something=foo`, for example).

## jsonx

> All of the examples in this section were generated against the data
> in [this document](https://gist.github.com/larsks/7519147).

`jsonx` is a tool for extracting data from a JSON response.  For
example, to obtain a token ID from a Keystone authentication response,
you could use:

    $ jsonx access/token/id < response.json

Which would get you:

    access/token/id b494ab5626ab4c65ab493ca427bce1b7

To get just the value, specify `-v`:

    $ jsonx -v access/token/id < response.json
    b494ab5626ab4c65ab493ca427bce1b7

You can use wildcards in your path expressions:

    $ jsonx access/serviceCatalog/*/type < response.json 
    access/serviceCatalog/0/type compute
    access/serviceCatalog/1/type network
    access/serviceCatalog/2/type s3
    access/serviceCatalog/3/type image
    access/serviceCatalog/4/type volume
    access/serviceCatalog/5/type ec2
    access/serviceCatalog/6/type object-store
    access/serviceCatalog/7/type identity

You can use `-k` to just show the keys in the result:

    $ jsonx -k access/* < response.json
    access/token
    access/serviceCatalog
    access/user
    access/metadata

You can use `-p` to pretty-print the output:

    $ jsonx -p access/token/tenant
    {
        "access": {
            "token": {
                "tenant": {
                    "enabled": true, 
                    "description": "admin tenant", 
                    "name": "admin", 
                    "id": "a4c4f915bf024710a1203f207aa3e4d3"
                }
            }
        }
    }

