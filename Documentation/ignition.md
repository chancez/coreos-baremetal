
# Ignition

Ignition is a system for declaratively provisioning disks from the initramfs, before systemd starts. It runs only on the first boot and handles formatting partitioning, writing files (systemd units, networkd units, dropins, regular files), and configuring users. See the Ignition [docs](https://coreos.com/ignition/docs/latest/) for details.

Ignition template files can be added in the `/var/lib/bootcfg/ignition` directory or in an `ignition` subdirectory of a custom `-data-path`. Template files may contain [Go template](https://golang.org/pkg/text/template/) elements which will be evaluated with Group `metadata` and should render to JSON or YAML (which will be served by `bootcfg` as JSON).

    /var/lib/bootcfg
     ├── cloud
     ├── ignition
     │   └── simple.json
     │   └── etcd.yaml
     │   └── etcd-proxy.yaml
     │   └── networking.yaml
     └── profiles

Reference an Ignition config in a [Profile](bootcfg.md#profiles). When PXE booting, use the kernel option `coreos.first_boot=1` and `coreos.config.url` to point to the `bootcfg` [Ignition endpoint](api.md#ignition-config).

## Configs

Here is an example Ignition template for static networking, which will be rendered, with metadata, into YAML and tranformed into machine-friendly JSON.

ignition/network.tmpl:

    ---
    ignition_version: 1
    networkd:
      units:
        - name: 00-{{.networkd_name}}.network
          contents: |
            [Match]
            Name={{.networkd_name}}
            [Network]
            Gateway={{.networkd_gateway}}
            DNS={{.networkd_dns}}
            Address={{.networkd_address}}
    {{ if .ssh_authorized_keys }}
    passwd:
      users:
        - name: core
          ssh_authorized_keys:
            {{ range $element := .ssh_authorized_keys }}
            - {{$element}}
            {{end}}
    {{end}}

Response from `/ignition?mac=address` for a particular machine.

    {
      "ignitionVersion": 1,
      "storage": {},
      "systemd": {},
      "networkd": {
        "units": [
          {
            "name": "00-ens3.network",
            "contents": "[Match]\nName=ens3\n[Network]\nGateway=172.15.0.1\nDNS=172.15.0.3\nDNS=8.8.8.8\nAddress=172.15.0.21/16\n"
          }
        ]
      },
      "passwd": {}
    }

Note that rendered Ignition does **not** allow variables - the response has been fully rendered with `metadata` for the requesting machine.

Ignition configs can be provided directly as JSON as well (`.ign` or `.ignition`). This is useful for simple cases or if you prefer to use your own templating solution to generate Ignition configs.

ignition/run-hello.ign:

    {
        "ignitionVersion": 1,
        "systemd": {
            "units": [
                {
                    "name": "hello.service",
                    "enable": true,
                    "contents": "[Service]\nType=oneshot\nExecStart=/usr/bin/echo Hello World\n\n[Install]\nWantedBy=multi-user.target"
                }
            ]
        }
    }

### Examples

See [examples/ignition](../examples/ignition) for example Ignition templates.

