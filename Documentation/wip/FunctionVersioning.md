# Use cases of creating versions of functions and packages

1. A user may want to save a history of all versions of source code deployed for a function, basically leaving an audit trail
2. A user may want to test different versions of his function ( canary testing )
3. A user may want to do a rolling upgrade from one version to another version of his function.
4. A user may want to submit his source code changes to git and then just invoke `fission spec version apply`, that'll take care of creating all of his functions, triggers, packages and environment objects.
5. A cluster administer may want a record of the exact versions of different functions deployed at any given point in time ( for troubleshooting some issues in the cluster and making them reproducible )

# Implementation proposal
On a high level 
1. This proposal intends to create a new function object every time the user does a `function version update` - basically creating a new function version.
   Implicitly, this means that fission takes care of assigning versions. 
2. The version numbers are not sequentially increasing numbers, instead, they are randomly generated uuid short strings. The reason for this is explained under "Implementation changes needed for fission spec".
3. So, every new function version will be a function object with name function name and a version uuid suffix
   For example - lets say a user first creates a function "fnA" with `fission function version create --name fnA` and then he updates this function with `fission fn version update --name fnA --code <new_code>`, a new function object is created with `Name: fnA-axv`.
   Next, when he updates the same function again, another new version of a function is created, ex : `Name: fnA-reh`.
3. This proposal also intends to version packages along with functions. This is because fundamentally, function versioning implies that the source code is versioned and in fission, packages contain source code. 
4. Naturally, the intention is to version packages similar to functions. So, every pkg update results in a new pkg object being created, which is treated as a version of that package.

# Implementation details
This proposal intends to create 3 new CRD object types for managing the state - `FunctionVersionTracker`, `PackageVersionTracker`, `GlobalVersionTracker`.

1. FunctionVersionTracker -  used to track all versions created for each function

2. PackageVersionTracker - used to track all versions created for each package

3. GlobalVersionTracker - used to track all objects created in each iteration of `spec version apply`. This will give the state of k8s cluster at that point in time. This is used for use-case 5 mentioned above.

(I'm sure you have a lot of questions at this point, but, reading further sections should answer most of them, if not all)

Classifying the implementation details broadly under two categories 
1. implementation needed W.R.T `fission spec` to automatically deploy all the desired objects from the spec into k8s cluster. 
2. implementation needed W.R.T fission cli commands where user individually creates the desired objects.

## 1. Implementation details for `fission spec`
Dividing this section into  - 
1. TL;DR walking you through various scenarios and how the code intends to behave and the snapshot of objects created etc. 
2. Pseudocode

### TL;DR of intended behavior of fission code through different scenarios
The list of scenarios are definitely not all-encompassing. but at a minimum, it's an attempt to show the generic behavior of versioning code.
Summary of the scenarios (CRUD) :
scenarios 1 - 4 touch upon creation of different versions of functions and packages.
scenarios 5 touch upon deletion of versions of function and packages.
There is no concept of updating versions - every new update on a function or a package results in a new version.
Listing all versions of a function and/or package is possible and that's covered under "Implementation details for CLI"

#### Version creation
1. For this scenario, let's assume there's nothing on the kubernetes cluster except that fission is installed and its micro-services are running.
   user's yaml contains the following :
   ```yaml
   apiVersion: fission.io/v1
    kind: Function
    metadata:
      name: hello
      namespace: default
    spec:
      environment:
        name: python-27
        namespace: default
      package:
        packageref:
          name: hello-pkg
          namespace: default
        functionName: hello.main
    
    ---
    apiVersion: fission.io/v1
    kind: Package
    metadata:
      name: hello-pkg
      namespace: default
    spec:
      source:
        url: archive://hello-archive
      buildcmd: "./build.sh"
      environment:
        name: python-27
        namespace: default
    status:
      buildstatus: pending
    
    ---
    kind: ArchiveUploadSpec
    name: hello-archive
    include:
      - "hello/*.py"
      - "hello/*.sh"
      - "hello/requirements.txt"
   ```
   Now, lets say, the user does a `fission spec version apply`, here's what will happen in the code :
   
   1. A function object with name function name with a version suffix, something like `hello-owf`, a package object with name `hello-pkg-owf` will get created on k8s cluster.
      in addition, objects of type `PackageVersionTracker`, `FunctionVersionTracker` and `GlobalVersionTracker` get created, here are the object snapshots :
      ```json
      {
            "kind": "FunctionVersionTracker",
            "metadata": {
                "name": "hello"
            },
            "spec": {
                "versions": ["owf"]
            } 
      },
      {
            "kind": "PackageVersionTracker",
            "metadata": {
                "name": "hello-pkg"
            },
            "spec": {
                "versions": ["owf"]
            } 
      },
      {
            "kind": "GlobalVersionTracker",
            "metadata": {
                "name": "owf"
            },
            "spec": {
                "git": {
                   "commit": "<git_commit_sha>",
                   "branch": "<git_branch_name>",
                   "clean": "yes"
                },
                "objects": [
                     {
                       "type": "function",
                       "name": "hello",
                       "version": "owf" // TODO : Comeback
                     },
                     {
                       "type": "package",
                       "name": "hello-pkg",
                       "version": "owf" // TODO : Comeback
                     }
                ]  
            } 
      },
      {
            "kind": "Function",
            "metadata": {
                "name": "hello-owf",
            },
            "spec": {
                "InvokeStrategy": {
                    "ExecutionStrategy": {
                        "ExecutorType": "poolmgr",
                        "MaxScale": 1,
                        "MinScale": 0,
                        "TargetCPUPercent": 80
                    },
                    "StrategyType": "execution"
                },
                "configmaps": [],
                "environment": {
                    "name": "python-27",
                    "namespace": "default"
                },
                "package": {
                    "packageref": {
                        "name": "hello-pkg-owf",
                        "namespace": "default",
                        "resourceversion": "101877"
                    }
                },
                "resources": {},
                "secrets": []
            }
      },
      { // TODO : combeback and make it sourcepkg
            "kind": "Package",
            "metadata": {
                "name": "hello-pkg-owf",
                "resourceversion": "101877"
            },
            "spec": {
                "deployment": {
                    "checksum": {},
                    "literal": "Cm1vZHVsZS5leHBvcnRzID0gYXN5bmMgZnVuY3Rpb24oY29udGV4dCkgewogICAgcmV0dXJuIHsKICAgICAgICBzdGF0dXM6IDIwMCwKICAgICAgICBib2R5OiAiSGVsbG8sIHdvcmxkIVxuIgogICAgfTsKfQo=",
                    "type": "literal"
                },
                "environment": {
                    "name": "python-27",
                    "namespace": "default"
                },
                "source": {
                    "checksum": {}
                }
            },
            "status": {
                "buildstatus": "succeeded"
            }
      }
      ```
   2. next, let's say the user changes the source code in one of the python files under "hello" directory and does a git commit, followed by a `fission spec version apply`.
   3. now the fission code creates a new archive object (this is not versioned); it sees that a package with the same name already exists, so it decides to version the package by creating a new package object with the same name suffixed by a version uid, ex: `hello-pkg-aub`
   ```json
       { // TODO : combeback and make it sourcepkg
            "kind": "Package",
            "metadata": {
                "name": "hello-pkg-aub",
                "resourceversion": "101879"
            },
            "spec": {
                "deployment": {
                    "checksum": {},
                    "literal": "Cm1vZHVsZS5leHBvcnRzID0gYXN5bmMgZnVuY3Rpb24oY29udGV4dCkgewogICAgcmV0dXJuIHsKICAgICAgICBzdGF0dXM6IDIwMCwKICAgICAgICBib2R5OiAiSGVsbG8sIHdvcmxkIVxuIgogICAgfTsKfQo=",
                    "type": "literal"
                },
                "environment": {
                    "name": "python-27",
                    "namespace": "default"
                },
                "source": {
                    "checksum": {}
                }
            },
            "status": {
                "buildstatus": "succeeded"
            }
       }
   ```
   4. the fission code also appends `versions` list in the `PackageVersionTracker` for this package `"hello-pkg"` with the newly created version uid :
   ```json
       {
            "kind": "PackageVersionTracker",
            "metadata": {
                "name": "hello-pkg"
            },
            "spec": {
                "versions": ["owf", "aub"]
            } 
       } 
   ```
   5. fission code will see that a function with the same name already exists and decides to version it by creating a new function object with name `"hello-aub"`:
   ```json
       {
            "kind": "Function",
            "metadata": {
                "name": "hello-aub",
            },
            "spec": {
                "InvokeStrategy": {
                    "ExecutionStrategy": {
                        "ExecutorType": "poolmgr",
                        "MaxScale": 1,
                        "MinScale": 0,
                        "TargetCPUPercent": 80
                    },
                    "StrategyType": "execution"
                },
                "configmaps": [],
                "environment": {
                    "name": "python-27",
                    "namespace": "default"
                },
                "package": {
                    "packageref": {
                        "name": "hello-pkg-aub",
                        "namespace": "default",
                        "resourceversion": "101879"
                    }
                },
                "resources": {},
                "secrets": []
            }
       },

   ```
   6. next, it also updates the versions list of `FunctionVersionTracker` for this function `"hello"` with the newly created version uid :
   ```json
       {
            "kind": "FunctionVersionTracker",
            "metadata": {
                "name": "hello"
            },
            "spec": {
                "versions": ["owf", "aub"]
            } 
       } 
   ```
   (at this point, please note that the version uid for both package and function objects are the same. TODO : Come back)
   7. the fission code will also create an object of type `GlobalVersionTracker`. this basically has data about all objects that were created/modified in this iteration of `spec version apply`.
   ```json
       {
            "kind": "GlobalVersionTracker",
            "metadata": {
                "name": "aub"
            },
            "spec": {
                "git": {
                   "commit": "<git_commit_sha>",
                   "branch": "<git_branch_name>",
                   "clean": "yes"
                },
                "objects": [
                     {
                       "type": "function",
                       "name": "hello",
                       "version": "aub" // TODO : Comeback
                     },
                     {
                       "type": "package",
                       "name": "hello-pkg",
                       "version": "aub" // TODO : Comeback
                     }
                ]  
            } 
       } 
   ```
   ( this might be the right point to explain why versions are not sequential numbers. let's assume the same scenario as above where the spec is checked into a git repo and many users concurrently create branches and work on their stuff.
   at any point in time the `spec version apply` might be called from anyone's branch. if we assigned sequential numbers, the numbers might go askew between branches and for a single branch, the numbers wont be contiguous. hence the ordering of versions is not maintained with numbers)
   8. At the end of scenario 1, all of the above objects are present on the cluster.

   
2. This scenario is similar to the first one, except we have an http trigger object in the spec for function `"hello""`. again starting with an empty k8s cluster with fission installed and the following yaml.
   This yaml is similar to scenario 1, only includes an http trigger object. I made this an entire scenario because it needs a little explanation with respect to function reference by version.
   ```yaml
   apiVersion: fission.io/v1
    kind: Function
    metadata:
      name: hello
      namespace: default
    spec:
      environment:
        name: python-27
        namespace: default
      package:
        packageref:
          name: hello-pkg
          namespace: default
        functionName: hello.main
    
    ---
    apiVersion: fission.io/v1
    kind: Package
    metadata:
      name: hello-pkg
      namespace: default
    spec:
      source:
        url: archive://hello-archive
      buildcmd: "./build.sh"
      environment:
        name: python-27
        namespace: default
    status:
      buildstatus: pending
    
    ---
    kind: ArchiveUploadSpec
    name: hello-archive
    include:
      - "hello/*.py"
      - "hello/*.sh"
      - "hello/requirements.txt"
    ---
    apiVersion: fission.io/v1
    kind: HTTPTrigger
    metadata:
      name: d2aa2980-71d5-40b4-a490-4969646cbf7f
      namespace: default
    spec:
      functionref:
        name: hello
        type: name
        version: latest
      host: ""
      method: GET
      relativeurl: /hello
    
   ``` 
   Now, lets say, the user does a `fission spec version apply`, here's what will happen in the code :
   
   1. a function object with name `hello`, a package object with name `hello-pkg` will get created on k8s cluster. also, an object of httptrigger is created with the given spec. notice that the `functionRef` has a `"version"` filed pointing to `"latest"` version of the function.
      this means that when the triggers are being resolved into the right function, it will map to the most recently created version of the function object. now, to find the most recently created version of the function, we use `FunctionVersionTracker` object which has a list of versions created for a function.
      Since we use a list, we automatically get the ordering, meaning, the last element in the list is the most recently created version of the function.
   2. now, lets say the user modifies his source code in one of the py files and invokes a spec version apply. Like explained in scenario, 2 function versions (`"hello"`, `"hello-aub"`) and 2 package ( `"hello-pkg"`, `"hello-pkg-aub"`) versions get created. however, there's no change to http trigger object. but because the version of the function it's pointing to is `"latest""`, at the time of function resolution, the trigger will be mapped to the `"hello-aub"`
   3. there may be a scenario where the user deletes a function version that was created most recently. in other words, the function object that had been resolved as `"latest"` will now need to point to the one created before that. so, the plan is have a functionVersionController that does a listWatch of `FunctionVersionTracker` object and constantly syncsTriggers to invalidate stale entries from router cache.
   3. if the explanation is too much to take in, please look at the code here ( TODO : paste a link to my commit ) : that should give you a rough idea of how the resolution will take place and a little about the `FunctionVersionController`
   
3. This scenario again starts with an empty k8s cluster with fission installed. While the user's yaml is similar to scenario 1, it has an additional function, package and archive. so basically showing the behavior of versioning when there are two functions with no shared packages.     
   ```yaml
   apiVersion: fission.io/v1
    kind: Function
    metadata:
      name: foo
      namespace: default
    spec:
      environment:
        name: python-27
        namespace: default
      package:
        packageref:
          name: foo-pkg
          namespace: default
        functionName: foo.main
    
    ---
    apiVersion: fission.io/v1
    kind: Package
    metadata:
      name: foo-pkg
      namespace: default
    spec:
      source:
        url: archive://foo-archive
      buildcmd: "./build.sh"
      environment:
        name: python-27
        namespace: default
    status:
      buildstatus: pending
    
    ---
    kind: ArchiveUploadSpec
    name: foo-archive
    include:
      - "foo/*.py"
      - "foo/*.sh"
      - "foo/requirements.txt"
    ---
    apiVersion: fission.io/v1
    kind: Function
    metadata:
      name: bar
      namespace: default
    spec:
      environment:
        name: python-27
        namespace: default
      package:
        packageref:
          name: bar-pkg
          namespace: default
        functionName: bar.main
    
    ---
    apiVersion: fission.io/v1
    kind: Package
    metadata:
      name: bar-pkg
      namespace: default
    spec:
      source:
        url: archive://bar-archive
      buildcmd: "./build.sh"
      environment:
        name: python-27
        namespace: default
    status:
      buildstatus: pending
    
    ---
    kind: ArchiveUploadSpec
    name: bar-archive
    include:
      - "bar/*.py"
      - "bar/*.sh"
      - "bar/requirements.txt"
    
   ``` 
   Now, lets say, the user does a `fission spec version apply`, here's what will happen in the code :
   
   1. two function objects with name `"foo"`, `"bar"`, two package objects with names `"foo-pkg"`, `"bar-pkg"` will get created on k8s cluster along with the `FunctionVersionTracker`, `PackageVersionTracker` and `GlobalVersionTracker` objects as explained in scenario 1.
   2. Now, lets say user updates the source code of only one of the functions, `"foo"` and invokes a `spec version apply`, fission code will see that package content varies only for `"foo"` and versions the function and package of `"foo"`.
   3. Here's a snap shot of all the objects in k8s cluster at this point :
   ```json
       {
            "kind": "FunctionVersionTracker",
            "metadata": {
                "name": "foo"
            },
            "spec": {
                "versions": ["yti", "abc"]
            } 
       },
       {
            "kind": "FunctionVersionTracker",
            "metadata": {
                "name": "bar"
            },
            "spec": {
                "versions": ["yti"]
            } 
       },
       {
            "kind": "PackageVersionTracker",
            "metadata": {
                "name": "foo-pkg"
            },
            "spec": {
                "versions": ["yti", "abc"]
            } 
       },
       {
            "kind": "PackageVersionTracker",
            "metadata": {
                "name": "bar-pkg"
            },
            "spec": {
                "versions": ["yti"]
            } 
       },
       {
            "kind": "GlobalVersionTracker",
            "metadata": {
                "name": "yti"
            },
            "spec": {
                "git": {
                   "commit": "<git_commit_sha>",
                   "branch": "<git_branch_name>",
                   "clean": "yes"
                },
                "objects": [
                     {
                       "type": "function",
                       "name": "foo",
                       "version": "yti"
                     },
                     {
                       "type": "function",
                       "name": "bar",
                       "version": "yti"
                     },
                     {
                       "type": "package",
                       "name": "foo-pkg",
                       "version": "yti" 
                     },
                     {
                       "type": "package",
                       "name": "bar-pkg",
                       "version": "yti" 
                     }
                ]  
            } 
       },
       {
            "kind": "GlobalVersionTracker",
            "metadata": {
                "name": "abc"
            },
            "spec": {
                "git": {
                   "commit": "<git_commit_sha>",
                   "branch": "<git_branch_name>",
                   "clean": "yes"
                },
                "objects": [
                     {
                       "type": "function",
                       "name": "foo",
                       "version": "abc"
                     },
                     {
                       "type": "function",
                       "name": "bar",
                       "version": "yti"
                     },
                     {
                       "type": "package",
                       "name": "foo-pkg",
                       "version": "abc" 
                     },
                     {
                       "type": "package",
                       "name": "bar-pkg",
                       "version": "yti" 
                     }
                ]  
            } 
       }
   ```
   please note that the `GlobalVersionTracker` object that gets created during the 2nd iteration of `spec version apply`, refers to older version of function and package for `"bar"`. this will be filled using `PackageVersionTracker` and `FunctionVersionTracker` objects to find out their `"latest"` versions. 
   It should be evident now that this object can be used by a cluster administer to reproduce issues and troubleshoot them. (use case 5)

4. The last scenario in version creation is one where there are 2 functions that share the same package, i.e. have the source code in one package for both functions. again, beginning with a fission installed k8s cluster with a clean slate. 
   user's yaml:
   ```yaml
   apiVersion: fission.io/v1
    kind: Function
    metadata:
      name: foo
      namespace: default
    spec:
      environment:
        name: python-27
        namespace: default
      package:
        packageref:
          name: src-pkg
          namespace: default
        functionName: foo.main
    
    ---
    apiVersion: fission.io/v1
    kind: Function
    metadata:
      name: bar
      namespace: default
    spec:
      environment:
        name: python-27
        namespace: default
      package:
        packageref:
          name: src-pkg
          namespace: default
        functionName: bar.main
    
    ---
    apiVersion: fission.io/v1
    kind: Package
    metadata:
      name: src-pkg
      namespace: default
    spec:
      source:
        url: archive://src-archive
      buildcmd: "./build.sh"
      environment:
        name: python-27
        namespace: default
    status:
      buildstatus: pending
    
    ---
    kind: ArchiveUploadSpec
    name: src-archive
    include:
      - "src/*.py"
      - "src/*.sh"
      - "src/requirements.txt"
    
   ``` 
   Now, lets say, the user does a `fission spec version apply`, here's what will happen in the code :
   
   1. two function objects with name `"foo-uid"`, `"bar-uid"`, one package object with names `"src-pkg-uid"` will get created on k8s cluster along with the `FunctionVersionTracker`, `PackageVersionTracker` and `GlobalVersionTracker` objects as explained earlier.
   2. Now, lets say user updates the function code of only one of the functions, `"foo"` and invokes a `spec version apply`, fission code will see that they share a package and hence version the package first and also version both functions. This might seem a little counter-intuitive, but its consistent with how `spec apply` would work.
   TODO : Verify this
   for ex : let's say there are two functions that refer a package in the spec and if the source code of one of the functions get updated in the package, then, other function gets updated too ( basically the package reference of the function spec ).
   Also, logically thinking, one of the reasons two functions may share a package is may be because one function talks to the other and they might have inter-dependent code. so, if one function's source is getting updated, there's a high chance the other function needs to be updated as well.
   3. Here's a snap shot of all the objects in k8s cluster at this point :
   ```json
       {
            "kind": "FunctionVersionTracker",
            "metadata": {
                "name": "foo"
            },
            "spec": {
                "versions": ["uid", "bla"]
            } 
       },
       {
            "kind": "FunctionVersionTracker",
            "metadata": {
                "name": "bar"
            },
            "spec": {
                "versions": ["uid", "bla"]
            } 
       },
       {
            "kind": "PackageVersionTracker",
            "metadata": {
                "name": "src-pkg"
            },
            "spec": {
                "versions": ["uid", "bla"]
            } 
       },
       {
            "kind": "GlobalVersionTracker",
            "metadata": {
                "name": "uid"
            },
            "spec": {
                "git": {
                   "commit": "<git_commit_sha>",
                   "branch": "<git_branch_name>",
                   "clean": "yes"
                },
                "objects": [
                     {
                       "type": "function",
                       "name": "foo",
                       "version": "uid"
                     },
                     {
                       "type": "function",
                       "name": "bar",
                       "version": "uid"
                     },
                     {
                       "type": "package",
                       "name": "foo-pkg",
                       "version": "uid" 
                     }
                ]  
            } 
       },
       {
            "kind": "GlobalVersionTracker",
            "metadata": {
                "name": "bla"
            },
            "spec": {
                "git": {
                   "commit": "<git_commit_sha>",
                   "branch": "<git_branch_name>",
                   "clean": "yes"
                },
                "objects": [
                     {
                       "type": "function",
                       "name": "foo",
                       "version": "bla"
                     },
                     {
                       "type": "function",
                       "name": "bar",
                       "version": "bla"
                     },
                     {
                       "type": "package",
                       "name": "src-pkg",
                       "version": "bla" 
                     }
                ]  
            } 
       }
   ```
   
 
#### Version deletion
Basically, with `spec`, its impossible to delete a specific version of a function or a package. This is because `spec` is designed to work like `kubectl apply`. So, the way an object in spec gets deleted is if its present in the spec at first and then removed from the spec, at which point, it's deleted.

so, if a user's yaml has a function in a particular iteration and it's missing in the next, then all versions created for that function will get deleted as part of the next `spec version apply` iteration.
Same applies to the packages. 

But fission cli can still be used to delete a particular version of a function. The spec stuff will work seamlessly even with deleted function versions or package versions.

Describing this explanation with a scenario :
5. For this scenario, let's continue from the state of k8s cluster at the end of scenario 4. To quickly summarize, the user's yaml had two functions, one package and one archive. The k8s cluster has two versions of functions `"foo"` and `"bar"` and two versions of package `"src-pkg"`.
   Now, if the user removes function foo from the users.yaml and also from the source code in the python files and invokes a `spec version apply`
   ```yaml
    apiVersion: fission.io/v1
    kind: Function
    metadata:
      name: bar
      namespace: default
    spec:
      environment:
        name: python-27
        namespace: default
      package:
        packageref:
          name: src-pkg
          namespace: default
        functionName: bar.main
    
    ---
    apiVersion: fission.io/v1
    kind: Package
    metadata:
      name: src-pkg
      namespace: default
    spec:
      source:
        url: archive://src-archive
      buildcmd: "./build.sh"
      environment:
        name: python-27
        namespace: default
    status:
      buildstatus: pending
    
    ---
    kind: ArchiveUploadSpec
    name: src-archive
    include:
      - "src/*.py"
      - "src/*.sh"
      - "src/requirements.txt"
    
   ```  
   1. then fission code will first create a new version of package because the contents of archive changed (devoid of source code related to function `"foo"`).
   2. so naturally, a new version of function `"bar"` gets created (because the package contents changed)
   3. finally, all function objects of `"foo"` get deleted, along with its `FunctionVersionTracker` object. 
   4. now, I thought about this quite a bit and finally decided it's not wise to delete the `GlobalVersionTracker` object that has references to function `"foo"` which doesnt exist in the system anymore. But, the whole point of having this object is to make some past build reproducible - it has git commit info, function versions of foo that existed at that time etc.
      (ymmv, glad to hear your opinion about this)
   5. finally, k8s cluster will have the following objects
   ```json
       {
            "kind": "FunctionVersionTracker",
            "metadata": {
                "name": "bar"
            },
            "spec": {
                "versions": ["uid", "bla", "ueo"]
            } 
       },
       {
            "kind": "PackageVersionTracker",
            "metadata": {
                "name": "src-pkg"
            },
            "spec": {
                "versions": ["uid", "bla", "ueo"]
            } 
       },
       {
            "kind": "GlobalVersionTracker",
            "metadata": {
                "name": "uid"
            },
            "spec": {
                "git": {
                   "commit": "<git_commit_sha>",
                   "branch": "<git_branch_name>",
                   "clean": "yes"
                },
                "objects": [
                     {
                       "type": "function",
                       "name": "foo",
                       "version": "uid"
                     },
                     {
                       "type": "function",
                       "name": "bar",
                       "version": "uid"
                     },
                     {
                       "type": "package",
                       "name": "foo-pkg",
                       "version": "uid" 
                     }
                ]  
            } 
       },
       {
            "kind": "GlobalVersionTracker",
            "metadata": {
                "name": "bla"
            },
            "spec": {
                "git": {
                   "commit": "<git_commit_sha>",
                   "branch": "<git_branch_name>",
                   "clean": "yes"
                },
                "objects": [
                     {
                       "type": "function",
                       "name": "foo",
                       "version": "bla"
                     },
                     {
                       "type": "function",
                       "name": "bar",
                       "version": "bla"
                     },
                     {
                       "type": "package",
                       "name": "src-pkg",
                       "version": "bla" 
                     }
                ]  
            } 
       },
       {
            "kind": "GlobalVersionTracker",
            "metadata": {
                "name": "ueo"
            },
            "spec": {
                "git": {
                   "commit": "<git_commit_sha>",
                   "branch": "<git_branch_name>",
                   "clean": "yes"
                },
                "objects": [
                     {
                       "type": "function",
                       "name": "bar",
                       "version": "ueo"
                     },
                     {
                       "type": "package",
                       "name": "src-pkg",
                       "version": "ueo" 
                     }
                ]  
            } 
       }
   ```
At this point, more scenarios can be added for `deletion`, but I'm not sure if my approach for deleting versions w.r.t `spec` is fundamentally wrong. Before I waste more time on thinking along these lines, I would like to hear your opinion.

### Pseudocode for spec
This pesudocode doesnt do justice to the exact code that would be needed, but just outlines the general functionality
```go
func specVersionApply() {
        // 1. generate a uniquie uid for this iteration of apply
        
        // 2. collect all the objects from the user's yaml into a global fissionResources struct (similar to specApply)
        
        // 3. apply individual objects starting from archives, packages, functions, env and triggers.
        for item := range objects {
                get existent list of objects of type item.Type
                
                if item already present and contents from deep equal are the same, do nothing.
                
                if item already present and contents differ from previous state {
                        1. if item.Type == Function or if item.Type == Package, create a new object of item
                        2. if item.Type == Function, create/update the `FunctionVersionTracker` for this functionName 
                        3. if item.Type == Package, create/update the `PackageVersionTracker` for this packageName 
                        4. update the `GlobalVersionTracker`
                }
                
                if item existed before but not present in the list of fissionResources in this iteration, then
                        if item.Type == Function, delete all versions of the function and delete `FunctionVersionTracker` obj for this functionName 
                        if item.Type == Package, delete all versions of the package and delete `PackageVersionTracker` obj for this packageName 
        }
}
```
## 2. Implementation details for fission cli commands
TODO : How it's similar to spec and how it's different/ more flexible or rigid from spec.

Similar to earlier, dividing this section into  - 
1. TL;DR walking you through various scenarios and how the code intends to behave and the snapshot of objects created etc. 
2. Pseudocode - for version create, delete, list

### TL;DR walking you through various scenarios
The list of scenarios are definitely not all-encompassing. but at a minimum, it's an attempt to show the generic behavior of versioning code.
Summary of the scenarios (CRUD) :
scenarios 1 - x touch upon creation of different versions of functions and packages.
scenarios y - z touch upon deletion of versions of function and packages.
There is no concept of updating versions - every new update on a function or a package results in a new version.
Listing all versions of a function and/or package is possible

1. Different versions of a function can be created with `fission function version create --name functionName --code hello.py`
   At this point, fission code creates a new function object with name functionName suffixed by randomly generated uuid.
   Also creates a `FunctionVersionTracker` object for this function.
   In this scenario, since fission cli creates a package for the code, it versions the pkg too with the same suffix and creates a `PackageVersionTracker` for this package.
   TODO : State of different objects created
   
2. `fission package version create` ,  so subsequent updates result in versioned packages.
   `fission function version create --pkg pkgName` -> creates function object with same version if pkg is versioned 
   `fission function version update --functionName --code` 
   `fission function version update --functionName --srcPackage` 
    
### Pseudocode
    

TODO :
1. think about labels, if we need any
2. update code for resolving function handler before pasting commit history.
3. also add the CRD types in the same commit.
4. go through scenarios for spec , for first iteration, fnNames and pkgNames shd have a uid.


