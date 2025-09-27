
```shell
python3 -m http.server 8000
```

```shell
sudo coreos-installer install /dev/sda --ignition-url=<url_ign_file> --insecure-ignition
```

```shell
docker run --interactive --rm --security-opt label=disable --volume "${PWD}:/pwd" --workdir /pwd quay.io/coreos/butane:release --pretty --strict config.bu > coreos.ign
```