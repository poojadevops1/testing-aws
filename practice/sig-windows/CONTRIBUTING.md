# Contributing on Windows Node support

Contains a list of common resources when contributing in the effort to support Windows Node and Windows Server containers in Kubernetes.

-   [Joining the community of other contributors](#joining-the-community-of-other-contributors)
-   [Find work in progress](#find-work-in-progress)
-   [Building Kubernetes for Windows from Source](#building-kubernetes-for-windows-from-source)
    -   [Build Prerequisites](#build-prerequisites)
    -   [Building Kubernetes binaries for Windows](#building-kubernetes-binaries-for-windows)
    -   [Updating the Node binaries](#updating-the-node-binaries)
-   [Creating a PR](#creating-a-pr)
-   [API Considerations](#api-considerations)
-   [Running Tests](#running-tests)
-   [Troubleshooting](#troubleshooting)
-   [Reporting Issues and Feature Requests](#reporting-issues-and-feature-requests)
-   [Gathering Logs](#gathering-logs)
    -   [Collecting Networking Logs](#collecting-networking-logs)

## Joining the community of other contributors

The best way to get in contact with the contributors working on Windows support is through the Kubernetes Slack. To get a Slack invite, visit [http://slack.k8s.io/](http://slack.k8s.io/) . Once you're logged in, join us in the [#SIG-Windows](https://kubernetes.slack.com/messages/sig-windows) channel.

To get access to shared documents, meeting calendar, and additional discussions, be sure to also join the [SIG-Windows Google Group](https://groups.google.com/forum/#!forum/kubernetes-sig-windows). 

View the leadership team in SIG-Windows and other subprojects in the [getting started](https://github.com/kubernetes/community/tree/master/sig-windows) guide.

## Find work in progress

View the SIG-Windows project boards:

- [Open Issues](https://github.com/orgs/kubernetes/projects/82)
- [Open PRs](https://github.com/orgs/kubernetes/projects/99)

Items are added to these boards automatically and these boards are reviewed during our [bi-weekly backlog refinement meeting](https://github.com/kubernetes/community/tree/master/sig-windows#meetings).

## Building Kubernetes for Windows from Source

The Kubernetes build scripts have not been ported to Windows, so it's best to develop in a Linux VM or [WSL2](https://docs.microsoft.com/windows/wsl/) environment where you can run the same Docker container used in the official Kubernetes builds.

It's best to read through the [Building Kubernetes](https://github.com/kubernetes/kubernetes/blob/master/build/README.md) guide if you have never built Kubernetes before to get the latest info. These steps are a summary focused on cross-building the Windows node binaries (kubectl, kubelet, and kube-proxy).

### Build Prerequisites

At least 60GB of disk space is required, and 16GB of memory (or memory + swap).

In your developer environment, install Git, [Docker-CE](https://docs.docker.com/install/), and make. The build scripts will pull a Docker container with the required version of golang and other needed tools preinstalled.

If you're using Ubuntu, then install the following packages: git, build-essential, [Docker-CE](https://docs.docker.com/install/linux/docker-ce/ubuntu/).

### Building Kubernetes binaries for Windows

You can build individual components such as kubelet, kube-proxy, or kubectl by running `./build/run.sh make <binary name> KUBE_BUILD_PLATFORMS=windows/amd64` such as `./build/run.sh make kubelet KUBE_BUILD_PLATFORMS=windows/amd64`

If you would like to build all binaries at once, then run `./build/run.sh make cross KUBE_BUILD_PLATFORMS=windows/amd64`

Once the build completes, the files will be in `_output/dockerized/bin`.

### Testing your changes

#### Build a local cluster

The easist way to test changes is to use the community maintained [SIG Windows developer environment](https://github.com/kubernetes-sigs/sig-windows-dev-tools#welcome-to-the-sig-windows-development-environment) project.
You can use this to build a fully functional cluster locally with binaries built from source.

#### Updating the Node binaries

If you already have an existing cluster you also test your changes by swapping the binaries on a node.
To update the binaries on an existing node, follow these steps:

1. Drain & cordon a node with `kubectl drain <nodename>`.
2. Connect to the node with SSH or Windows Remote Desktop, and start PowerShell
3. On the node, run `Stop-Service kube-proxy -Force` followed by `Stop-Service kubelet -Force`.
4. Copy kubelet.exe and kube-proxy.exe to the node.
5. Overwrite the existing kubelet & kube-proxy binaries. If you don't know where they are, run `sc.exe qc kubelet` or `sc.exe qc kube-proxy` and look at the BINARY_PATH_NAME returned.
6. Start the updated kubelet & kube-proxy with `Start-Service kubelet`.

## Creating a PR

Congratulations on contributing to the SIG-Windows ecosystem.
If there is a PR you would like to build, it's easy. You can create a working branch, pull the changes from GitHub in a patch, apply, then build.

Please follow the Kubernetes contribute guide for [pull-requests](https://github.com/kubernetes/community/blob/master/contributors/guide/pull-requests.md) when submitting your changes.

Additionally please perform the following:

1. Add the **sig/windows** label to your PR by adding `/sig windows` in the pull request description or a comment.
1. Trigger windows specific e2e test by adding `/test pull-kubernetes-e2e-aks-engine-windows-containerd` in a comment.

## API Considerations

If you modifying an API in the SIG-Windows codebase, make sure you are aware of the API [guidelines](/contributors/devel/sig-architecture/api_changes.md) and [conventions](/contributors/devel/sig-architecture/api-conventions.md) used in Kubernetes. This [document offers guidelines for API reviewers that API developers should always have in consideration](https://docs.google.com/document/d/1pkCYjr_OLCRUk2e606eMa2PEIwJdFHAfU5slXtBcKj8/edit).

## Running Tests

For the most up-to-date steps on how to build and run tests, please go to [https://github.com/kubernetes-sigs/windows-testing](https://github.com/kubernetes-sigs/windows-testing). It has everything you need to build and run tests, as well as links to the SIG-Windows configurations used on [TestGrid](https://testgrid.k8s.io/sig-windows).  

Specifically, you need to build the e2e test binary, using https://github.com/kubernetes-sigs/windows-testing/blob/master/README.md#how-do-i-build-the-e2etest-binary, in order to run these tests.

## Troubleshooting

### Intro

If you are having issues with network dependent services coming up with your Windows based container there is a workaround that may help.

We can use the [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) to help ensure the service starts after the network is available.  Specifically we will be using the [PostStart](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks) hook.

### Examples

The first example will execute after the pod is up and in this particular case will restart the `dbconnect` service once `dbhost.example.com` can be resolved via DNS.  This command could also be modified to require said host to be reachable before exiting.

```
        lifecycle:
          postStart:
            exec:
              command: ["powershell.exe","-command","do { $Result = @(ping -n 1 dbhost.example.com) } while ( $Result -notcontains 'Approximate round trip times in milli-seconds:' ); Restart-Service -Name dbconnect"]
```

This second example is used in pods where GMSA is being used.  This will restart the `netlogon` service until we can affirm that the pod has logged on to the domain correctly.

```
        lifecycle:
          postStart:
            exec:
              command: ["powershell.exe","-command","do { Restart-Service -Name netlogon } while ( $($Result = (nltest.exe /query); if ($Result -like '*0x0 NERR_Success*') {return $true} else {return $false}) -eq $false)"]
```

## Reporting Issues and Feature Requests

If you have what looks like a bug, or you would like to make a feature request, please use the [Github issue tracking system](https://github.com/kubernetes/kubernetes/issues). You can open issues on [GitHub](https://github.com/kubernetes/kubernetes/issues/new/choose) and assign them to SIG-Windows. You should first search the list of issues in case it was reported previously and comment with your experience on the issue and add additional logs. SIG-Windows Slack is also a great avenue to get some initial support and troubleshooting ideas prior to creating a ticket.

If filing a bug, please include detailed information about how to reproduce the problem, such as:

* Kubernetes version: kubectl version
* Environment details: Cloud provider, OS distro, networking choice and configuration, and Docker version
* Detailed steps to reproduce the problem
* Relevant logs
* Tag the issue sig/windows by commenting on the issue with `/sig windows` to bring it to a SIG-Windows member's attention

## Gathering Logs

Logs are an important element of troubleshooting issues in Kubernetes. Make sure to include them any time you seek troubleshooting assistance from other contributors.

### Collecting kubelet and kube-proxy Logs
There are a few different ways to run the node binaries (kubelet, kube-proxy, etc) and the execution method also dictates how logs will be collected. [Issue 75319](https://github.com/kubernetes/kubernetes/issues/75319) tracks pending work for better log management on Windows (for example using the Windows Event Log for higher log throughput and log rotation). If you end up logging to a file, you can use [fluentd](https://docs.fluentd.org/v0.12/articles/windows) or [Splunk](https://docs.splunk.com/Documentation/Splunk/7.2.4/Data/HowtogetWindowsdataintoSplunk) to ship logs to a syslog server for search and analytics.
1. Windows Service Manager services - You may have to introduce your own log consumption and log rotation service if you are logging to a file. We will investigate if journald is an option
2. nssm.exe services - nssm.exe provides support for forwarding stdout/stderr logs to a file (use the `AppStdout` and `AppStderr` options) and also supports logs file rotation (See the `File rotation` and `I/O redirection` sections in the [documentation](https://nssm.cc/usage)). You can see more examples on using nssm.exe for the Kubernetes components of the Windows node in the services and background processes section under [troubleshooting](https://kubernetes.io/docs/getting-started-guides/windows/#troubleshooting)
```Powershell
# Example nssm command line for the kubelet
nssm set kubelet AppStdout C:\k\kubelet.log
nssm set kubelet AppStderr C:\k\kubelet.log
```

### Collecting Networking Logs
1. On the node before creating the pod for the first time.
2. `start-bitstransfer https://raw.githubusercontent.com/Microsoft/SDN/master/Kubernetes/windows/debug/collectlogs.ps1`
3. Execute collectlogs.ps1 in a PowerShell window
4 Start the trace by running `C:\k\debug\starthnstrace.cmd`
5. Reproduce the issue
6. Run `netsh trace stop`
7. Execute collectlogs.ps1 in a PowerShell window again
8. Include in your ticket `C:\server.etl`
