## Part 1

> What does a Deployment give you that a bare Pod doesn't? Why do we need a separate Service object instead of talking to a Pod's IP directly?

If a bare pod crashes or the node it's on goes away, it doesn't come back. We'd have to notice and recreate it by hand. However, a Deployment wraps a Pod template in a control loop that continuously reconciles the desired replica count against reality: it restarts crashed Pods, reschedules them onto healthy nodes, and gives us rolling updates/rollbacks when we change the Pod spec.

We need a separate Service because a Pod's IP is not stable since every time a Pod is recreated (by the Deployment's control loop, after a crash, a reschedule, a rollout), it gets a new IP. If other Pods hardcoded that IP, the connection would break on the next restart. However, a Service puts a fixed address in front of all the Pods matching a given label, and spreads traffic across whichever of those Pods happen to be alive at the moment. That way, anything calling into the Service is relying on a name that stays constant even as the individual Pods behind it come and go.

## Part 2

> What is a Helm chart, and how do `values.yaml` plus `-f`/`--set` let the same chart produce different Kubernetes resources for different environments? Why is this better than keeping separate copies of your Part 1 manifests for staging and production?

A Helm chart is a package of Kubernetes manifests written as templates (in `templates/`), plus a `values.yaml` file of default settings. Instead of hardcoding names, images, or resource sizes, the templates reference `.Values.*` and `.Release.Name`. Installing the chart under a release name renders the templates with a concrete set of values and applies the result as a tracked, versioned release.

`values.yaml` supplies the defaults. At install or upgrade time, `-f <file>.yaml` merges overrides on top, and `--set key=value` overrides everything. Because every environment-specific setting (model name, resource limits, storage size) is a value rather than a literal, the same templates can be rendered with `values-staging.yaml` and `values-production.yaml` to produce two differently sized sets of resources. These are installed as separate releases (`ollama-staging` and `ollama-production`). Since resource names come from `.Release.Name`, the two releases don't collide, and each can be upgraded or rolled back independently.

This is better than keeping separate copies of the Part 1 manifests because there is only one copy of the logic to maintain. With duplicated manifests, every fix or structural change has to be repeated by hand in each copy, and the copies easily drift out of sync. With a chart, the change is made once in `templates/`, and every environment picks it up on its next `helm upgrade`. Only the environment-specific values differ, not the structure of the resources, and Helm's release history also makes rollbacks simple.
