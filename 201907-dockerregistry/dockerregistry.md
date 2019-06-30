# Deploy a private docker registry

In a previous blog post we have setup a NVIDIA Jetson Nano powered K8s cluster. We are now ready to build some containerized app/service on top of it. Unlike x86/amd64 world, there are less off the shelf containers built for arm/arm64 and I often found myself having to build it from scratch. To deploy the in-house built containers, you will first need a private docker registry (unless you'd like to push everything to docker hub).

In this post, I'd like to cover the steps how to build a private [docker registry](https://docs.docker.com/registry/) that came with [a neat web UI](https://github.com/Joxit/docker-registry-ui) that can be used to search and clean up images.

There are 3 steps you will need to do:
- Generate a CA cert
- Compose a container set with a docker registry and an UI server
- Add CA to other boxes trusted list

## References
