
```shell
python3 -m http.server 8000
```

```shell
sudo coreos-installer install /dev/sda --ignition-url=<url_ign_file> --insecure-ignition
```

In editor, convert generated .ign file into UTF-8 to remove BOM characters


```shell
docker run --interactive --rm --security-opt label=disable --volume "${PWD}:/pwd" --workdir /pwd quay.io/coreos/butane:release --pretty --strict k8s-config.bu > coreos.ign
```