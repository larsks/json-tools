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

## Examples

### Authenticate to Keystone and getting an acess token

    #!/bin/sh

    # pass the ip address or hostname of your keystone server
    # on the command line.
    keystone_url=http://$1:5000/v2.0

    token=$(
      jsong auth/passwordCredentials/username=admin \
        auth/passwordCredentials/password=secret \
        auth/tenantName=admin |
      curl -s -H 'content-type: application/json' \
        --data-binary @- ${keystone_url}/tokens |
      jsonx -v access/token/id
    )

    echo "got token: $token"

### Get a list of subnets from Quantum

For each network available in Quantum, this will print the network
name and a list of subnet ids and their corresponding cidr range.

    #!/bin/sh

    # pass the ip address or hostname of your keystone server
    # on the command line.
    keystone_url=http://$1:5000/v2.0

    work=$(mktemp -d jsonXXXXXX)
    trap "rm -rf $work" EXIT

    jsong auth/passwordCredentials/username=admin \
      auth/passwordCredentials/password=secret \
      auth/tenantName=admin |
    curl -s -H 'content-type: application/json' \
      --data-binary @- ${keystone_url}/tokens > $work/token.json

    token=$(jsonx -v access/token/id < $work/token.json)

    network_path=$(
      jsonx access/serviceCatalog/*/type < $work/token.json | 
      awk '$2 == "network" {print $1}'
      )

    network_url=$(
      jsonx -v ${network_path%/type}/endpoints/0/publicURL < $work/token.json
      )

    curl -s \
      -H 'content-type: application/json' \
      -H "x-auth-token: $token" \
      $network_url/v2.0/networks > $work/networks.json

    for network in $(jsonx -k networks/* < $work/networks.json); do
      jsonx -v $network/name < $work/networks.json

      for subnet in $(jsonx -v ${network}/subnets/* < $work/networks.json); do
        curl -s \
          -H 'content-type: application/json' \
          -H "x-auth-token: $token" \
          $network_url/v2.0/subnets/$subnet > $work/subnet-$subnet.json

        echo "  $subnet $(jsonx -v subnet/cidr < $work/subnet-$subnet.json)"
      done
    done

