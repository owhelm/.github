## owhelm

owhelm is for when you're overwhelmed with Helm.

### Rationale

Helm is missing several critical features:

- calculated values
- post-renderers that are shipped together with the chart

Getting them into Helm itself is worthwhile, but it might not be easy - and that is understandable for a piece of Open Source Software (OSS) which has a huge user base. Large OSS projects have huge inertia and they also have serious responsibilities to maintain compatibility for a very diverse set of edge cases. Moreover, both of the above features can change the threat model and enlarge the vulnerability surface.

owhelm could try to implement these features as a plugin/layer/wrapper of Helm, making it opt-in, and with time and luck permitting - grow into a standard via proof of usefulness by adoption and battle testing to uncover edge cases.

### Use cases

#### For calculated values

- simplified value re-use - having variables is the cornerstone of all software engineering, yet Helm does not have that. Helm does allow some code re-use via Go template helpers, however this is limited - it is only usable in templates - and so it requires developers to use template helpers (i.e. functions) for nearly everything.
    - e.g. a chart uses multiple images. For a typical deployment, one might want to override the registry (to go via local proxy). A chart could still accept a `.Values.specificComponent.image.registry`, which falls back to `.Values.image.registry` (specific to the chart), which could further fall back to `.Values.global.image.registry` (share-able across all sub-charts).
    - e.g. `tolerations` and `affinity` may need to be both specified for a `Pod`. They typically reference the same node selectors and could re-use a single value to reference them.
- as a chart developer, reduce the need to call `tpl` for every use `.Values` in a chart.
    - e.g. sharing a domain name across multiple contexts when the user of the chart needs to have it set as a label/annotation as well as a primary configuration value.
- passing the values downstream, to sub-charts.
    - e.g. as a parent chart consumer, setting the domain name only once.
 
#### For baked-in post-renderers

- for cross-cutting concerns in wrapper charts
    - e.g. to be able to set `tolerations` and `affinity`, the sub-chart needs to expose them as acceptable values and the consumer needs to make sure to set these values as many times as there are different Pods. A post-renderer could set them based on a single value instead without requiring a sub-chart to expose the primitives.
- for keeping the logic encapsulated and the same as it moves across environments
    - e.g. continuing with the `tolerations` / `affinity` example, if post-renderer is baked into the same tarball of the chart, then the logic can change with the application version, allowing the same logic to be re-used in all environments without deploying additional software.

### Existing alternatives

- admission hooks allow mutation of resources
    - con: requires a cluster-wide deployment of something like Kyverno or an implementation of an HTTP server
    - con: does not leverage (and plays against) the diffing functionality of tools like Argo CD
- `--post-renderer`
    - con: Helm only passes through the final rendered resources as a stream/buffer (i.e. a string), but not the original values or context (e.g. command or release information)
    - con: the tool itself needs to be pre-installed wherever `helm` is running
- `Helmfile`
    - con: no distribution mechanism; external to the chart itself
- Using a tool to generate the `values.yaml`
    - con: no common standard, in-house domain specific software
    - con: external to the chart, and therefore has a different release cycle, which is potentially incompatible with supporting multiple release lines of the same chart
