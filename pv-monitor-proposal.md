# PV monitoring proposal

Status: Pending

Version: Alpha

Implementation Owner: NickrenREN@ 

## Motivation

For now, kubernetes has no way to monitor the PVs, which may cause serious problems. 
For example: if volumes are unhealthy, and pods do not know that and still try to get and write data, 
which will lead to data loss and unavailability of services. 
So it is necessary to have a mechanism for monitoring PVs and react when PVs have problems.

## Proposal

We can separate the proposal into two parts:

* monitoring PVs and marking them if they have problems
* reacting to the unhealthy PVs

For monitoring, we may create a controller for it, and each volume plugin should have its own function to check volume health. 
Controller can call them periodically.

For reacting, different kinds of apps may have different methods,we can also create a controller for it. 

At first phase, we may just react to statefulsets with local storage PVs.

## User Experience
### Use Cases

* Users want to know if the volumes their apps are referencing are healthy or not;
* We should mark the PV if it is not healthy;
* Controllers such as statefulset controller should be aware of the unhealthy volumes and react to them;

## Implementation

As mentioned above, we can split this into two parts and put them in the external repo at first.

### Monitoring controller: 

Like PV controller, monitoring controller should check PVs’ health condition periodically and taint them if PVs are unhealthy.
Health checking implementation should be per plugin. Each volume plugin needs to have its own methods to check its volumes.

#### For in-tree volume plugins(except local storage):

We can add a new volume plugin interface: StatusCheckingVolumePlugin.
```
  type StatusCheckingVolumePlugin interface {
       VolumePlugin

      CheckStatus(spec *Spec) (string, error)
  }
```
And each volume plugin will implement it. The entire monitoring controller workflow is:

* Fill PV cache with initial data from etcd
* Resync and check volumes status periodically
* Taint PV if the volume status is abnormal
* Remove the taints if the volume becomes normal again

#### For in-tree volume plugins(except local storage):

We can create a new controller called MonitorController like ProvisionController, 
which is responsible for creating informers and calling each plugin’s monitor functions. 
And each volume plugin will create its own monitor to check its volumes’ status.

#### For in-tree volume plugins(except local storage):

We can put monitor in its provisioner package and each provisioner is responsible for checking volumes health condition in that specific node.

At the first phase, we can support local storage monitoring first.

Take local storage as an example, implementation may be like this:

```
 func (mon *minitor) CheckStatus(spec *v1.PersistentVolumeSpec) (string, error) {
            // check if PV is local storage
    if pv.Spec.Local == nil { 
		glog.Infof("PV: %s is not local storage", pv.Name)
		return ...
	}
            // check if PV belongs to this node
	if checkNodeAffinity(...) {
		glog.Infof("PV: %s does not belong to this node", pv.Name)
		return ...
	}

            // check if host dir still exists
               ...
	
	// check ...

}
```
If monitor finds that one PV is unhealthy, it will mark the PV.(adding annotations)

TBD: should we mark the PV when we find out that its status is not normal for the first time, 
or mark the PV if the PV is found to be abnormal for several times.

### Reaction controller:

Reaction controller will react to the PV update event (tainting PV by monitoring controller). 
Different kinds of apps should have different reactions.

At the first phase, we just consider the statefulset reaction.

Statefulset reaction: delete the PVC bound to the unhealthy volume(PV) as well as pods referencing it.

Reaction controller’s workflow is:

* Fill PV cache from etcd;
* Watch for PV update events;
* Resync and populator periodically;
* Delete related PVC and pods if PV is found unhealthy(just for statefulsets and reclaim PV depending on reclaim policy);

### PV controller changes:
For unbound PVCs/PVs,  PVCs will not be bound to PVs which have taints.

## Roadmap to support PV monitoring

* support local storage PV monitoring(mark PVs), and just react to statefulsets
* support other out-tree volume plugins and add PV taint support
* support in-tree volume plugins and react to other kinds of applications if needed

## Alternatives considered

