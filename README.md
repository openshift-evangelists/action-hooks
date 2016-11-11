# Action Hooks

OpenShift 2 had a concept of [action hooks](https://developers.openshift.com/managing-your-applications/action-hooks.html) built-in to many of the language cartridges. These came in two flavours, cartridge control action hooks and build action hooks. The name 'build action hooks' was actually a misnomer as they didn't apply to just the build phase with there also being specific action hooks related to the deployment of the application as well.

Of the two types of action hooks the latter was the most important as it allowed an end user to add extra actions to be performed when their application code was being added to OpenShift, as well as when it was being started up. It didn't provide complete control, which was only available by writing a custom cartridge of your own, but it was still sufficient for doing many tasks.

Depending on the language and framework being used, the action hooks feature was the only way to add in extra steps. Despite its usefulness, there is no direct equivalent action hooks feature in OpenShift 3. This can especially be an issue for Python due to there being so many different methods for deploying Python web applications. You need the extra customisation abilities to be able to properly set up and configure some Python web applications or web servers.

Implementing the equivalent of action hooks with the new [Source-to-Image](https://github.com/openshift/source-to-image) (S2I) builders can be done, but it is no where as clean as being an in built feature. There is also various issues one has to deal with depending on what S2I builder you are using and for which you need the feature.

This code repository is intended as a example of how you can integrate a more structured action hooks mechanism into your source code repository and have the S2I process trigger it when doing builds and deployments. The action hooks mechanism described is similar in some respects to what OpenShift 2 had, but also provides other hook points based on experience from deploying web applications over many years to OpenShift, Heroku and other hosting services.

## Implementing Action Hooks

The way to implement action hooks yourself is to provide custom S2I ``assemble`` and ``run`` scripts as part of the source code for your application. These will run the additional actions at the appropriate time and then invoke the original ``assemble`` and ``run`` scripts.

The location in the source code for your application where these custom S2I scripts need to reside is the ``.s2i/bin`` directory under the top level directory of your application. This would normally be the top level directory of your source code directory, but could also be in a sub directory if the ``--context-dir`` option were being used with ``oc new-app`` or ``oc build`` when creating a deployment or build.

The basic structure of a replacement ``assemble`` script would be:

    #!/bin/bash
 
    # Do stuff you need to do before original assemble script.
    
    # Run the original assemble script.
    
    $S2I_SCRIPTS_PATH/assemble
    
    # Do stuff you need to do after the original assemble script.

Couple of issues here you need to be aware of.

The first issue is that if doing stuff before the original assemble script, the source code of your application will not have been copied into place, compiled, or whatever it is the original ``assemble`` script did.

If what you need to do before the original ``assemble`` script needs files from your application source code, it will need to use them out of the ``/tmp/src`` directory. You might consider copying files from ``/tmp/src`` into place yourself, but you have to be very careful of doing that, as some S2I builders might delete directories you create as they don't merge stuff when copying and instead move directories/files thus overwriting any you create before the original ``assemble`` script is run.

The second issue is knowing where the original ``assemble`` script is located. There is no consistency between S2I builders as to where they are located and no best practice recommendation as to where they should be put. There is also no environment variable that all S2I builders use to define the location of the original S2I scripts.

Indications are that there are no plans to standardise on the use of an environment variable to define the location of the original S2I scripts, you will therefore need to hardwire it into any custom ``assemble`` or ``run`` scripts. Alternatively, if you are writing generic scripts you might use in multiple projects using different S2I builders, you can set your own environment variable in the ``.s2i/environment`` file which specifies the location. In the example scripts shown here we assume a ``S2I_SCRIPTS_PATH`` environment variable has been set.

The basic structure of a replacement ``run`` script would be:

    #!/bin/bash
    
    # Do stuff you need to do before original run script.
    
    # Run the original run script.
    
    exec $S2I_SCRIPTS_PATH/run

Note that when running the original ``run`` script it must be done using ``exec`` so that the original ``run`` script inherits process ID 1. This is important so that the final application run in the container is still process ID 1 and correctly receives signals sent to the container.

## Types of Action Hooks

Under OpenShift 2, the action hooks which were provided were:

* **pre_build** - Executed prior to building application artefacts.
* **build** - Executed after building application artefacts.
* **deploy** - Executed prior to starting the application.
* **post_deploy** - Executed after the application has been started.

Equivalent hooks to all but **post_deploy** can readily be implemented. The reason that **post_deploy** cannot be implemented is that when using Docker, the application is generally left running as process ID 1. That is, no process manager is usually used within a running Docker container.

That there is no overarching process manager makes it somewhat difficult to run something after the application has been started as the application actually keeps control and is not running in background. Solutions such as running **post_deploy** in the background, delayed by a sleep, isn't practical as you can't  be sure the application has actually been started properly before running it.

If required, anything like **post_deploy** is therefore better implemented outside of the container, using features of OpenShift such as lifecycle hooks.

As well as not implementing **post_deploy**, personal experience from working in Python suggests that the action hooks are better off modelled a bit differently with additional functionality added. For the purpose of this discussion though we will keep reasonably close to the original in OpenShift 2 as it isn't possible to break open the original S2I ``assemble`` and ``run`` scripts to insert additional hook points.

Two new action hooks that will be added though are **build_env** and **deploy_env**. Technically these aren't hook scripts in the way the existing OpenShift 2 actions are. This is because they will not be executed as a distinct process, but inline to the replacement ``assemble`` and ``run`` scripts. Their purpose is to allow additional environment variables to be dynamically set. This can be important when needing to set environment variables dynamically based on information extracted from packages installed as part of the build process.

## Sample Implementation

A working sample implementation of action hooks can be found in this repository:

* [https://github.com/getwarped/action-hooks](https://github.com/getwarped/action-hooks)

To make use of this implementation, first copy the following files into your own application source code repository:

* .s2i/bin/assemble
* .s2i/bin/run

They should be added at the same location, creating the directories as necessary.

The next step is to create the file ``.s2i/environment``. This is going to be used to set an additional environment variables so that the replacement S2I scripts know where your original S2I scripts were located.

If using the Python S2I builder, you should add to ``.s2i/environment`` the variable:

    S2I_SCRIPTS_PATH=/usr/libexec/s2i

An additional environment variable can also be set. This should only be set where the S2I builder being used, uses a specific hack to try and enable any SCL packages it requires for any shell session.

In the case of the Python S2I builder, what the builder defines as part of the image are the following environment variables.

    BASH_ENV=/opt/app-root/etc/scl_enable
    ENV=/opt/app-root/etc/scl_enable
    PROMPT_COMMAND=". /opt/app-root/etc/scl_enable"

What these do is that any time a new shell session is created, the script file at ``/opt/app-root/etc/scl_enable`` will be executed. This script will enable any SCL packages by adjusting the ``PATH`` and setting the ``LD_LIBRARY_PATH`` environment variable to include the directories where SCL package shared libraries are located.

Unfortunately that script also then unsets all of these environment variables, which means that an ``assemble`` script doesn't know where that magic script is. We want to know the location so we can add additional code to it which will also trigger the **deploy_env** action hook, so that a shell session will always get exactly the same environment as would be set up when an application is being run. If this isn't done, then any script run manually by a user may fail due to having a differing set of environment variables.

You need therefore to work out if this hack is used by a S2I builder image and set the ``S2I_BASH_ENV`` environment variable so that the override ``assemble`` script knows where this file is.

    S2I_BASH_ENV=/opt/app-root/etc/scl_enable

## Using the Action Hooks

To now add your own action hooks, create the following files as necessary:

* .s2i/action_hooks/pre_build
* .s2i/action_hooks/build_env
* .s2i/action_hooks/build
* .s2i/action_hooks/deploy_env
* .s2i/action_hooks/deploy

The ``pre_build``, ``build`` and ``deploy`` scripts must all be executable. This is necessary due to a [bug](https://github.com/docker/docker/issues/9547) in Docker support for some file systems. It is not possible for the ``assemble`` script to do ``chmod +x`` on scripts prior to running. If you forget the implementation of actions hooks provided will warn you.

The ``pre_build``, ``build`` and ``deploy`` scripts would normally be shell scripts, but could technically be any executable program you can run to do what you need. If using a shell script, it is recommended to set:

    set -eo pipefail
    
so that the scripts will fail fast, with an error propagated back up to the ``assemble`` or ``run`` script. You can print out messages from these scripts if necessary to help debugging.

The ``build_env`` and ``deploy_env`` scripts must be shell scripts. They do not need to be executable nor have a ``#!`` line. They will be executed inline to the ``assemble`` and ``run`` scripts, being interpreted as a ``bash`` script.

These two scripts can be used to set any environment variables you need to set. It is not necessary to export variables as any variables set in the scripts will be automatically exported. Being evaluated as a shell script, you can include shell logic or use inline parameter substitution. You can thus do things like:

    LOGLEVEL=${LOGLEVEL:-1}

Just keep in mind that if including complicated logic that requires temporary variables, that they will be automatically exported. You may wish to use shell functions and bash local variables to restrict what is exported to whatever is set at global scope.

You should not print any messages from ``deploy_env`` as that will be executed for any shell session and the output may interfere with the result when running one off commands using ``oc exec`` or ``oc rsh``.


## Seeing it in Action

You can see how the action hooks mechanism works by deploying this source code repository as an application under OpenShift 3 using the command:

```
oc new-app https://github.com/getwarped/action-hooks
```

This will deploy a small Python web application. You can check the build logs and logs for the pod to see progress messages as everything is being run, including output from the sample action hook scripts provided.
