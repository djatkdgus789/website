---
reviewers:
title: 스케줄링 프레임워크
content_type: concept
weight: 70
---

<!-- overview -->

{{< feature-state for_k8s_version="v1.15" state="alpha" >}}

스케줄링 프레임워크는 쿠버네티스 스케줄러를 위한 착탈식(pluggable) 아키텍처이다.
새로운 "플러그인" API 집합을 기존 스케줄러에 추가하여 스케줄러를 쉽게 
사용자 정의할 수 있다. 플러그인은 스케줄러로 컴파일된다. API는
대부분의 스케줄링 기능이 플러그인으로 구현되는 것을 허용하면서 
스케줄링 "코어"는 간단하고 유지관리할 수 있게 유지할 수 있다. 프레임워크 디자인에 대해서 
더 많은 기술적 정보는 [스케줄링 프레임워크의 디자인 제안][kep]을 참고하자.

[kep]: https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md




<!-- body -->

# 프레임워크 워크플로우

스케줄링 프레임워크는 몇 가지 확장점을 정의한다. 스케줄러 플러그인은
스케줄러 플러그인은 하나 이상의 확장점에서 호출될 수 있도록 등록한다. 플러그인들 중 일부는 
스케줄링 결정을 변경할 수 있고 일부는 정보만 제공할 수 있다. 
The Scheduling Framework defines a few extension points. Scheduler plugins
register to be invoked at one or more extension points. Some of these plugins
can change the scheduling decisions and some are informational only.

파드를 스케줄하는 과정은 **스케줄링 사이클** 과 **바인딩 사이클**로 구분된다.
Each attempt to schedule one Pod is split into two phases, the **scheduling
cycle** and the **binding cycle**.

## 스케줄링 사이클 & 바인딩 사이클

스케줄링 사이클은 파드에 노드를 선정한다. 그리고 바인징 사이클은 결정을 
클러스터에 적용한다. 스케줄링 사이클과 바인딩 사이클은 합쳐 "스케줄링 컨텍스트"라고 한다.
The scheduling cycle selects a node for the Pod, and the binding cycle applies
that decision to the cluster. Together, a scheduling cycle and binding cycle are
referred to as a "scheduling context".

스케줄링 사이클들은 순차적으로 실행되는 반면, 바인딩 사이클들은 동시에 실행될 수 있다.
Scheduling cycles are run serially, while binding cycles may run concurrently.

스케줄링 사이클이나 바인딩 사이클은 파드가 스케줄이 불가능하거나 
내부적인 오류가 있을 경우 중단될 수 있다. 파드는 
큐로 돌아와 재시도된다. 
A scheduling or binding cycle can be aborted if the Pod is determined to
be unschedulable or if there is an internal error. The Pod will be returned to
the queue and retried.

## 확장점

다음 그림은 파드의 스케줄링 컨텍스트와 스케줄링 프레임워크가 
노출하는 확장점을 보여준다. 이 그림에서 "필터"는 
"Predicate"과 동일하고 "스코어링"은 "우선순위 함수"와 동일하다.
The following picture shows the scheduling context of a Pod and the extension
points that the scheduling framework exposes. In this picture "Filter" is
equivalent to "Predicate" and "Scoring" is equivalent to "Priority function".

하나의 플러그인은 더 복잡하거나 stateful 작업을 수행하기 위해서 여러개의 확장점에 등록이 가능하다.
One plugin may register at multiple extension points to perform more complex or
stateful tasks.

{{< figure src="/images/docs/scheduling-framework-extensions.png" title="scheduling framework extension points" >}}

### QueueSort {#queue-sort}

These plugins are used to sort Pods in the scheduling queue. A queue sort plugin
essentially provides a `Less(Pod1, Pod2)` function. Only one queue sort
plugin may be enabled at a time.

### PreFilter {#pre-filter}

These plugins are used to pre-process info about the Pod, or to check certain
conditions that the cluster or the Pod must meet. If a PreFilter plugin returns
an error, the scheduling cycle is aborted.

### Filter

These plugins are used to filter out nodes that cannot run the Pod. For each
node, the scheduler will call filter plugins in their configured order. If any
filter plugin marks the node as infeasible, the remaining plugins will not be
called for that node. Nodes may be evaluated concurrently.

### PostFilter {#post-filter}

These plugins are called after Filter phase, but only when no feasible nodes
were found for the pod. Plugins are called in their configured order. If
any postFilter plugin marks the node as `Schedulable`, the remaining plugins
will not be called. A typical PostFilter implementation is preemption, which
tries to make the pod schedulable by preempting other Pods.

### PreScore {#pre-score}

These plugins are used to perform "pre-scoring" work, which generates a sharable
state for Score plugins to use. If a PreScore plugin returns an error, the
scheduling cycle is aborted.

### Score {#scoring}

These plugins are used to rank nodes that have passed the filtering phase. The
scheduler will call each scoring plugin for each node. There will be a well
defined range of integers representing the minimum and maximum scores. After the
[NormalizeScore](#normalize-scoring) phase, the scheduler will combine node
scores from all plugins according to the configured plugin weights.

### NormalizeScore {#normalize-scoring}

These plugins are used to modify scores before the scheduler computes a final
ranking of Nodes. A plugin that registers for this extension point will be
called with the [Score](#scoring) results from the same plugin. This is called
once per plugin per scheduling cycle.

For example, suppose a plugin `BlinkingLightScorer` ranks Nodes based on how
many blinking lights they have.

```go
func ScoreNode(_ *v1.pod, n *v1.Node) (int, error) {
    return getBlinkingLightCount(n)
}
```

However, the maximum count of blinking lights may be small compared to
`NodeScoreMax`. To fix this, `BlinkingLightScorer` should also register for this
extension point.

```go
func NormalizeScores(scores map[string]int) {
    highest := 0
    for _, score := range scores {
        highest = max(highest, score)
    }
    for node, score := range scores {
        scores[node] = score*NodeScoreMax/highest
    }
}
```

If any NormalizeScore plugin returns an error, the scheduling cycle is
aborted.

{{< note >}}
Plugins wishing to perform "pre-reserve" work should use the
NormalizeScore extension point.
{{< /note >}}

### Reserve {#reserve}

A plugin that implements the Reserve extension has two methods, namely `Reserve`
and `Unreserve`, that back two informational scheduling phases called Reserve
and Unreserve, respectively. Plugins which maintain runtime state (aka "stateful
plugins") should use these phases to be notified by the scheduler when resources
on a node are being reserved and unreserved for a given Pod.

The Reserve phase happens before the scheduler actually binds a Pod to its
designated node. It exists to prevent race conditions while the scheduler waits
for the bind to succeed. The `Reserve` method of each Reserve plugin may succeed
or fail; if one `Reserve` method call fails, subsequent plugins are not executed
and the Reserve phase is considered to have failed. If the `Reserve` method of
all plugins succeed, the Reserve phase is considered to be successful and the
rest of the scheduling cycle and the binding cycle are executed.

The Unreserve phase is triggered if the Reserve phase or a later phase fails.
When this happens, the `Unreserve` method of **all** Reserve plugins will be
executed in the reverse order of `Reserve` method calls. This phase exists to
clean up the state associated with the reserved Pod.

{{< caution >}}
The implementation of the `Unreserve` method in Reserve plugins must be
idempotent and may not fail.
{{< /caution >}}

### Permit

_Permit_ plugins are invoked at the end of the scheduling cycle for each Pod, to
prevent or delay the binding to the candidate node. A permit plugin can do one of
the three things:

1.  **approve** \
    Once all Permit plugins approve a Pod, it is sent for binding.

1.  **deny** \
    If any Permit plugin denies a Pod, it is returned to the scheduling queue.
    This will trigger the Unreserve phase in [Reserve plugins](#reserve).

1.  **wait** (with a timeout) \
    If a Permit plugin returns "wait", then the Pod is kept in an internal "waiting"
    Pods list, and the binding cycle of this Pod starts but directly blocks until it
    gets approved. If a timeout occurs, **wait** becomes **deny**
    and the Pod is returned to the scheduling queue, triggering the
    Unreserve phase in [Reserve plugins](#reserve).

{{< note >}}
While any plugin can access the list of "waiting" Pods and approve them
(see [`FrameworkHandle`](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/20180409-scheduling-framework.md#frameworkhandle)), we expect only the permit
plugins to approve binding of reserved Pods that are in "waiting" state. Once a Pod
is approved, it is sent to the [PreBind](#pre-bind) phase.
{{< /note >}}

### PreBind {#pre-bind}

These plugins are used to perform any work required before a Pod is bound. For
example, a pre-bind plugin may provision a network volume and mount it on the
target node before allowing the Pod to run there.

If any PreBind plugin returns an error, the Pod is [rejected](#reserve) and
returned to the scheduling queue.

### Bind

These plugins are used to bind a Pod to a Node. Bind plugins will not be called
until all PreBind plugins have completed. Each bind plugin is called in the
configured order. A bind plugin may choose whether or not to handle the given
Pod. If a bind plugin chooses to handle a Pod, **the remaining bind plugins are
skipped**.

### PostBind {#post-bind}

This is an informational extension point. Post-bind plugins are called after a
Pod is successfully bound. This is the end of a binding cycle, and can be used
to clean up associated resources.

## Plugin API

There are two steps to the plugin API. First, plugins must register and get
configured, then they use the extension point interfaces. Extension point
interfaces have the following form.

```go
type Plugin interface {
    Name() string
}

type QueueSortPlugin interface {
    Plugin
    Less(*v1.pod, *v1.pod) bool
}

type PreFilterPlugin interface {
    Plugin
    PreFilter(context.Context, *framework.CycleState, *v1.pod) error
}

// ...
```

## Plugin configuration

You can enable or disable plugins in the scheduler configuration. If you are using
Kubernetes v1.18 or later, most scheduling
[plugins](/docs/reference/scheduling/config/#scheduling-plugins) are in use and
enabled by default.

In addition to default plugins, you can also implement your own scheduling
plugins and get them configured along with default plugins. You can visit
[scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins) for more details.

If you are using Kubernetes v1.18 or later, you can configure a set of plugins as
a scheduler profile and then define multiple profiles to fit various kinds of workload.
Learn more at [multiple profiles](/docs/reference/scheduling/config/#multiple-profiles).

