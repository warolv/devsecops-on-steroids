# How to instal falco

## Add falcosecurity repo
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

## Install falco from values.yaml to falco namespace
helm install falco falcosecurity/falco --namespace falco \
--create-namespace -f ./values.yaml