# Anarcho-Tech NYC: Lighttpd [![Build Status](https://travis-ci.org/AnarchoTechNYC/ansible-role-lighttpd.svg?branch=master)](https://travis-ci.org/AnarchoTechNYC/ansible-role-lighttpd)

An [Ansible role](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html) for installing a [Lighttpd](http://radicale.org/) server. Notably, this role has been tested with [Raspbian](https://www.raspbian.org/) on [Raspberry Pi](https://www.raspberrypi.org/) hardware. This role's purpose is to make it simple to install and run a Web server with minimal resources.

## Role variables

To configure Lighttpd for a simple use case, several convenience variables are supplied:

* `lighttpd_server_username`: The user account under which Lighttpd will run. Defaults to `www-data`.
* `lighttpd_server_home_dir`: The filesystem path of the Lighttpd account's root directory. Defaults to `/var/www`.
* `lighttpd_server_modules`: List of [Lighttpd modules to load](https://redmine.lighttpd.net/projects/lighttpd/wiki/Server_modulesDetails). Defaults to `["mod_access", "mod_alias", "mod_compress", "mod_redirect"]`.
* `lighttpd_server_document_root`: The Web [document root of the Lighttpd server](https://redmine.lighttpd.net/projects/lighttpd/wiki/Server_document-rootDetails). Defaults to `{{ lighttpd_server_home_dir }}/html`.
* `lighttpd_server_errorlog`: Filesystem path to [log runtime errors to](https://redmine.lighttpd.net/projects/lighttpd/wiki/Server_errorlogDetails). Defaults to `/var/log/lighttpd/error.log`.
* `lighttpd_server_port`: [Port number to listen for connections on](https://redmine.lighttpd.net/projects/lighttpd/wiki/Server_portDetails). Defaults to `80`.
* `lighttpd_server_index_file_names`: List of [files to serve if a directory listing is requested](https://redmine.lighttpd.net/projects/lighttpd/wiki/Index-file-names_Details). Defaults to `["index.php", "index.html", "index.lighttpd.html"]`.

## Advanced Lighttpd configuration

This role features a relatively thorough Jinja template implementation of [the Lighttpd configuration file format syntax](https://redmine.lighttpd.net/projects/lighttpd/wiki/Docs_Configuration#BNF-like-notation-of-the-basic-syntax). This means you can configure Lighttpd using the `lighttpd_config` list, whose items are dictionaries (two-dimensional associative arrays) that map nearly one-to-one to the [Lighttpd module names and their keys](https://redmine.lighttpd.net/projects/lighttpd/wiki/Docs_ConfigurationOptions#Configuration-File-Options). This implementation supports arbitrarily deeply nested [conditionals](https://redmine.lighttpd.net/projects/lighttpd/wiki/Docs_Configuration#Conditional-Configuration), Lighttpd arrays, and simple [variables](https://redmine.lighttpd.net/projects/lighttpd/wiki/Docs_Configuration#Using-variables).

The most complex part of the implementation is conditionals, which use the keyword `conditional` as the Lighttpd option list item name. The item itself is a list of dictionaries, each with the following required keys:

* `field`
* `operator`
* `value`
* `options`

All of these except `options` are described on the Lighttpd documentation for [conditional configuration](https://redmine.lighttpd.net/projects/lighttpd/wiki/Docs_Configuration#Conditional-Configuration). The `options` item is itself a list of Lighttpd options that are to be applied if the conditional is matched. This list can include another `conditional`.

Note that Lighttpd is strict with regard to quote style. It exclusively uses double quotes, not single quotes. This means you should be careful to use proper quoting when defining the `lighttpd_config` list in your vars files.

Examples should help make this clear:

1. Simple server bound to the alternate HTTP port, 8080.
    ```yaml
    lighttpd_config:
      - server:
          port: 8080
      - server:
          document-root: "{{ lighttpd_server_home_dir }}/html"
    ```
    The above is equivalent to the Lighttpd configuration file:
    ```
    server.port = 8080
    server.document-root: "/var/www/html"
    ```
1. Simple virtual host configuration with the default document root different from a domain-specific document root:
    ```yaml
    lighttpd_config:
      - server:
          document-root: "{{ lighttpd_server_home_dir }}/html"
      - conditional:
        - field: '$HTTP["host"]'
          operator: '=~'
          value: '(^|\.)example\.com$' # Single quotes to protect `\.`, which is a Jinja2 escape sequence.
          options:
            - server:
                document-root: "{{ lighttpd_server_home_dir }}/example.com"
    ```
    The above is equivalent to the Lighttpd configuration file:
    ```
    server.document-root = "/var/www/html"
    $HTTP["host"] =~ "(^|\.)example\.com$" {
        server.document-root = "/var/www/html/example.com"
    }
    ```

See the [`defaults/main.yaml`](defaults/main.yaml) file for a more thorough example.
