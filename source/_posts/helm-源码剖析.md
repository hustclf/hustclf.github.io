---
title: helm 源码剖析
tags:
  - helm
  - k8s
categories:
  - 云原生
abbrlink: 413a7d5a
date: 2022-02-13 17:34:08
---
## 背景
helm 是非常流行的 k8s 应用管理工具。类似于 python 中的 pip，我们使用 Chart 来定义一个 k8s 应用，使用 helm 来进行应用的安装、升级、发布和回滚。
本文旨在对 helm (v3) 的工作原理进行剖析，通过代码走查了解 helm 执行的具体过程，当使用 helm 出现问题时能更容易地定位。 （helm v2 的架构是 cli + server 的组合，已废弃）
源码的版本是[v3.8.0](https://github.com/helm/helm/tree/v3.8.0)。

阅读本文的一些背景知识：
- k8s 的基本概念：如 Deployment、 Service
- golang 的基础知识
- helm 的基本使用

<!-- more -->

## helm 的工作原理
如果使用一句话总结 helm 的工作原理，那就是 helm 是 k8s 的 pip。 我们都对python 的包管理工具 pip 很熟悉，因此可以类比来理解 helm 的功能。 如下图所示：
- pip 来定义和打包 python package
- docker cli 来定义和打包 container
- helm 来定义和打包 k8s application

<img src="/images/helm/1.png">

pip 和 helm 的功能对比
<img src="/images/helm/2.png">

pip 和 helm 的基本操作对比
<img src="/images/helm/3.png">


大致的工作原理如下：
- 仅仅是个 CLI:helm 不存储任何数据, charts 包保存在本机，helm 生成的 release 保存在 k8s 集群
- 主要逻辑是完成 templates 和 values 的渲染，并且请求 k8s apiserver 执行。
-<img src="/images/helm/4.png">

（I have templates, I have a values.yaml, Ugh, k8s Objects.）

## helm 的基本操作
使用 helm -h 可查看具体用法如下：
```
Usage:
helm [command]

Available Commands:
completion  generate autocompletion scripts for the specified shell
create      create a new chart with the given name
dependency  manage a chart's dependencies
env         helm client environment information
get         download extended information of a named release
help        Help about any command
history     fetch release history
install     install a chart
lint        examine a chart for possible issues
list        list releases
package     package a chart directory into a chart archive
plugin      install, list, or uninstall Helm plugins
pull        download a chart from a repository and (optionally) unpack it in local directory
repo        add, list, remove, update, and index chart repositories
rollback    roll back a release to a previous revision
search      search for a keyword in charts
show        show information of a chart
status      display the status of the named release
template    locally render templates
test        run tests for a release
uninstall   uninstall a release
upgrade     upgrade a release
verify      verify that a chart at the given path has been signed and is valid
version     print the client version information
```


接下来主要介绍下 install rollback upgrade  uninstall 等的具体实现。

## helm 各模块的代码实现
helm 的代码结构如下。helm 是使用cobra 实现的cli，其中主要的3个目录功能如下：

- cmd: cli 的入口，定义了各子命令的入参和执行入口
- pkg： 各子命令的具体实现
- internal： 内部共用的utils
```
├── cmd
│   └── helm
├── internal
│   ├── fileutil
│   ├── ignore
│   ├── monocular
│   ├── resolver
│   ├── sympath
│   ├── test
│   ├── third_party
│   ├── tlsutil
│   ├── urlutil
│   └── version
├── pkg
│   ├── action
│   ├── chart
│   ├── chartutil
│   ├── cli
│   ├── downloader
│   ├── engine
│   ├── gates
│   ├── getter
│   ├── helmpath
│   ├── kube
│   ├── lint
│   ├── plugin
│   ├── postrender
│   ├── provenance
│   ├── pusher
│   ├── registry
│   ├── release
│   ├── releaseutil
│   ├── repo
│   ├── storage
│   ├── strvals
│   ├── time
│   └── uploader
```

### 入口逻辑
cmd/helm/helm.go 是 程序入口，主要逻辑有如下两块：

1. newRootCmd（cmd/helm/helm.go:66）， 其主要做了如下工作
- helm 及子命令和参数注册
- k8sClient、RegistryClient 的初始化
- helm 插件的加载

2. `cobra.OnInitialize（cmd/helm/helm.go:73）`， 其主要为每个命令执行前预设了前置工作，主要是配置的初始化。
```go
    // run when each command's execute method is called
    cobra.OnInitialize(func() {
        helmDriver := os.Getenv("HELM_DRIVER")
        if err := actionConfig.Init(settings.RESTClientGetter(), settings.Namespace(), helmDriver, debug); err != nil {
            log.Fatal(err)
        }
        if helmDriver == "memory" {
            loadReleasesInMemory(actionConfig)
        }
    })
```

### helm install
helm install 负责初始化一个 Release。

执行入口是cmd/helm/install.go， 该文件包含3个函数：
- newInstallCmd：install 子cmd 的初始化
- addInstallFlags： install 命令的参数注册
- runInstall： `helm install` 的执行入口

runInstall 为 install 具体执行进行参数的初始化，最终由pkg/action/install.go的 RunWithContext 执行具体的安装工作，
确定的参数如下：
- release name
- char name
- dependencies
- values

RunWithContext 负责最终执行安装的动作，其主要工作如下：

- 预安装依赖的 charts （如果有）
- 预安装 CRDs， crds 定义在 Chart 的 `crd/` 下 （一些 Chart 可能定义了 CRD）
- 渲染 Chart 和 values.yaml, 生成最终的 k8s Objects
- 请求 k8s，将生成的k8s Objects 生成，并将 helm 的 release 信息保持到 k8s。
- 一些旁路逻辑：如是否是dryRun，是否在install 后执行 tests 的内容、钩子动作的注册等。
具体细节可参考: `pkg/action/install.go:189`
```go
// Run executes the installation with Context
func (i *Install) RunWithContext(ctx context.Context, chrt *chart.Chart, vals map[string]interface{}) (*release.Release, error) {
	// Check reachability of cluster unless in client-only mode (e.g. `helm template` without `--validate`)
	if !i.ClientOnly {
		if err := i.cfg.KubeClient.IsReachable(); err != nil {
			return nil, err
		}
	}

	if err := i.availableName(); err != nil {
		return nil, err
	}

	if err := chartutil.ProcessDependencies(chrt, vals); err != nil {
		return nil, err
	}

	// Pre-install anything in the crd/ directory. We do this before Helm
	// contacts the upstream server and builds the capabilities object.
	if crds := chrt.CRDObjects(); !i.ClientOnly && !i.SkipCRDs && len(crds) > 0 {
		// On dry run, bail here
		if i.DryRun {
			i.cfg.Log("WARNING: This chart or one of its subcharts contains CRDs. Rendering may fail or contain inaccuracies.")
		} else if err := i.installCRDs(crds); err != nil {
			return nil, err
		}
	}

	if i.ClientOnly {
		// Add mock objects in here so it doesn't use Kube API server
		// NOTE(bacongobbler): used for `helm template`
		i.cfg.Capabilities = chartutil.DefaultCapabilities.Copy()
		if i.KubeVersion != nil {
			i.cfg.Capabilities.KubeVersion = *i.KubeVersion
		}
		i.cfg.Capabilities.APIVersions = append(i.cfg.Capabilities.APIVersions, i.APIVersions...)
		i.cfg.KubeClient = &kubefake.PrintingKubeClient{Out: ioutil.Discard}

		mem := driver.NewMemory()
		mem.SetNamespace(i.Namespace)
		i.cfg.Releases = storage.Init(mem)
	} else if !i.ClientOnly && len(i.APIVersions) > 0 {
		i.cfg.Log("API Version list given outside of client only mode, this list will be ignored")
	}

	// Make sure if Atomic is set, that wait is set as well. This makes it so
	// the user doesn't have to specify both
	i.Wait = i.Wait || i.Atomic

	caps, err := i.cfg.getCapabilities()
	if err != nil {
		return nil, err
	}

	// special case for helm template --is-upgrade
	isUpgrade := i.IsUpgrade && i.DryRun
	options := chartutil.ReleaseOptions{
		Name:      i.ReleaseName,
		Namespace: i.Namespace,
		Revision:  1,
		IsInstall: !isUpgrade,
		IsUpgrade: isUpgrade,
	}
	valuesToRender, err := chartutil.ToRenderValues(chrt, vals, options, caps)
	if err != nil {
		return nil, err
	}

	rel := i.createRelease(chrt, vals)

	var manifestDoc *bytes.Buffer
	rel.Hooks, manifestDoc, rel.Info.Notes, err = i.cfg.renderResources(chrt, valuesToRender, i.ReleaseName, i.OutputDir, i.SubNotes, i.UseReleaseName, i.IncludeCRDs, i.PostRenderer, i.DryRun)
	// Even for errors, attach this if available
	if manifestDoc != nil {
		rel.Manifest = manifestDoc.String()
	}
	// Check error from render
	if err != nil {
		rel.SetStatus(release.StatusFailed, fmt.Sprintf("failed to render resource: %s", err.Error()))
		// Return a release with partial data so that the client can show debugging information.
		return rel, err
	}

	// Mark this release as in-progress
	rel.SetStatus(release.StatusPendingInstall, "Initial install underway")

	var toBeAdopted kube.ResourceList
	resources, err := i.cfg.KubeClient.Build(bytes.NewBufferString(rel.Manifest), !i.DisableOpenAPIValidation)
	if err != nil {
		return nil, errors.Wrap(err, "unable to build kubernetes objects from release manifest")
	}

	// It is safe to use "force" here because these are resources currently rendered by the chart.
	err = resources.Visit(setMetadataVisitor(rel.Name, rel.Namespace, true))
	if err != nil {
		return nil, err
	}

	// Install requires an extra validation step of checking that resources
	// don't already exist before we actually create resources. If we continue
	// forward and create the release object with resources that already exist,
	// we'll end up in a state where we will delete those resources upon
	// deleting the release because the manifest will be pointing at that
	// resource
	if !i.ClientOnly && !isUpgrade && len(resources) > 0 {
		toBeAdopted, err = existingResourceConflict(resources, rel.Name, rel.Namespace)
		if err != nil {
			return nil, errors.Wrap(err, "rendered manifests contain a resource that already exists. Unable to continue with install")
		}
	}

	// Bail out here if it is a dry run
	if i.DryRun {
		rel.Info.Description = "Dry run complete"
		return rel, nil
	}

	if i.CreateNamespace {
		ns := &v1.Namespace{
			TypeMeta: metav1.TypeMeta{
				APIVersion: "v1",
				Kind:       "Namespace",
			},
			ObjectMeta: metav1.ObjectMeta{
				Name: i.Namespace,
				Labels: map[string]string{
					"name": i.Namespace,
				},
			},
		}
		buf, err := yaml.Marshal(ns)
		if err != nil {
			return nil, err
		}
		resourceList, err := i.cfg.KubeClient.Build(bytes.NewBuffer(buf), true)
		if err != nil {
			return nil, err
		}
		if _, err := i.cfg.KubeClient.Create(resourceList); err != nil && !apierrors.IsAlreadyExists(err) {
			return nil, err
		}
	}

	// If Replace is true, we need to supercede the last release.
	if i.Replace {
		if err := i.replaceRelease(rel); err != nil {
			return nil, err
		}
	}

	// Store the release in history before continuing (new in Helm 3). We always know
	// that this is a create operation.
	if err := i.cfg.Releases.Create(rel); err != nil {
		// We could try to recover gracefully here, but since nothing has been installed
		// yet, this is probably safer than trying to continue when we know storage is
		// not working.
		return rel, err
	}
	rChan := make(chan resultMessage)
	go i.performInstall(rChan, rel, toBeAdopted, resources)
	go i.handleContext(ctx, rChan, rel)
	result := <-rChan
	//start preformInstall go routine
	return result.r, result.e
}
```


### helm upgrade
helm upgrade 负责更新当前的 Release，生成一个新版本的 Release。

同入install 比较类似，upgrade的执行入口是cmd/helm/upgrade.go， 不过所有功能都放在 newUpgradeCmd函数中

upgrade cmd 的初始化
参数注册
更新依赖的charts到本地
绑定执行入口，并最终执行 pkg/action/upgrade.go的 RunWithContext 执行具体的更新工作。
upgrade.go的 RunWithContext 的主要工作如下：

- 从获取前一次的 release 信息，并确保上次的release 不是 pendding 状态
- 向k8s 更新 denpendencies chats
- 渲染 Chart 和 values.yaml, 生成最终的 k8s Objects
- 一些旁路逻辑：如是否是dryRun，是否在 upgrade 后执行 tests 的内容、钩子动作的注册等。
具体细节可参考: `pkg/action/upgrade.go:130`
```go
// RunWithContext executes the upgrade on the given release with context.
func (u *Upgrade) RunWithContext(ctx context.Context, name string, chart *chart.Chart, vals map[string]interface{}) (*release.Release, error) {
    if err := u.cfg.KubeClient.IsReachable(); err != nil {
        return nil, err
    }

	// Make sure if Atomic is set, that wait is set as well. This makes it so
	// the user doesn't have to specify both
	u.Wait = u.Wait || u.Atomic

	if err := chartutil.ValidateReleaseName(name); err != nil {
		return nil, errors.Errorf("release name is invalid: %s", name)
	}
	u.cfg.Log("preparing upgrade for %s", name)
	currentRelease, upgradedRelease, err := u.prepareUpgrade(name, chart, vals)
	if err != nil {
		return nil, err
	}

	u.cfg.Releases.MaxHistory = u.MaxHistory

	u.cfg.Log("performing update for %s", name)
	res, err := u.performUpgrade(ctx, currentRelease, upgradedRelease)
	if err != nil {
		return res, err
	}

	if !u.DryRun {
		u.cfg.Log("updating status for upgraded release for %s", name)
		if err := u.cfg.Releases.Update(upgradedRelease); err != nil {
			return res, err
		}
	}

	return res, nil
}
```


### helm rollback
helm rollback 负责将当前的 Release 回滚至上一版本。

rollback的执行入口是 cmd/helm/rollback.go, 其中包含 newRollbackCmd，有以下功能：

rollback 子cmd 的初始化
参数注册
注册rollback 的具体回滚函数 Run，定义在 pkg/action/rollback.go:58
rollback.go 的 Run的主要工作如下：
- 从k8s 中获取当前的 Release 和回滚的Release
- 请求 k8s，将当前版本的Release回滚为目标 Release。 这里的实现略复杂，会对两个 release 进行 diff，并进行更新
- 请求 k8s 更新 Release 信息
 一些旁路逻辑： 如是否dryRun, 是否执行钩子、是否需要等待 rollback 结束等等
具体细节可参考: `pkg/action/rollback.go:58`
```go
// Run executes 'helm rollback' against the given release.
func (r *Rollback) Run(name string) error {
    if err := r.cfg.KubeClient.IsReachable(); err != nil {
        return err
    }

	r.cfg.Releases.MaxHistory = r.MaxHistory

	r.cfg.Log("preparing rollback of %s", name)
	currentRelease, targetRelease, err := r.prepareRollback(name)
	if err != nil {
		return err
	}

	if !r.DryRun {
		r.cfg.Log("creating rolled back release for %s", name)
		if err := r.cfg.Releases.Create(targetRelease); err != nil {
			return err
		}
	}

	r.cfg.Log("performing rollback of %s", name)
	if _, err := r.performRollback(currentRelease, targetRelease); err != nil {
		return err
	}

	if !r.DryRun {
		r.cfg.Log("updating status for rolled back release for %s", name)
		if err := r.cfg.Releases.Update(targetRelease); err != nil {
			return err
		}
	}
	return nil
}
```


### helm uninstall
helm uninstall 负责将当前的 Release 从 k8s 集群卸载掉。 helm uninstall 默认会删除 helm install 执行生成的所有k8s objects，也可配置保留部分资源。
rollback的执行入口是 cmd/helm/uninstall.go, 其中包含 newUninstallCmd，有以下功能：
uninstall子cmd 的初始化
参数注册
注册 uninstall 的具体回滚函数 Run，定义在 pkg/action/uninstall.go:54
uninstall.go 的 Run的主要工作如下：

从k8s 中获取所有的 Release 信息。
- 获取当前的 Release 信息，并请求 k8s 删除其关联的 k8s objects
- 根据 `keep-history` 决定是否要从 k8s 中删除 Release 的历史记录
- 一些旁路逻辑： 如是否dryRun, 是否执行钩子、是否需要等待 uninstall结束等

具体细节可参考：`pkg/action/uninstall.go:54`

```go
// Run uninstalls the given release.
func (u *Uninstall) Run(name string) (*release.UninstallReleaseResponse, error) {
    if err := u.cfg.KubeClient.IsReachable(); err != nil {
        return nil, err
    }

	if u.DryRun {
		// In the dry run case, just see if the release exists
		r, err := u.cfg.releaseContent(name, 0)
		if err != nil {
			return &release.UninstallReleaseResponse{}, err
		}
		return &release.UninstallReleaseResponse{Release: r}, nil
	}

	if err := chartutil.ValidateReleaseName(name); err != nil {
		return nil, errors.Errorf("uninstall: Release name is invalid: %s", name)
	}

	rels, err := u.cfg.Releases.History(name)
	if err != nil {
		return nil, errors.Wrapf(err, "uninstall: Release not loaded: %s", name)
	}
	if len(rels) < 1 {
		return nil, errMissingRelease
	}

	releaseutil.SortByRevision(rels)
	rel := rels[len(rels)-1]

	// TODO: Are there any cases where we want to force a delete even if it's
	// already marked deleted?
	if rel.Info.Status == release.StatusUninstalled {
		if !u.KeepHistory {
			if err := u.purgeReleases(rels...); err != nil {
				return nil, errors.Wrap(err, "uninstall: Failed to purge the release")
			}
			return &release.UninstallReleaseResponse{Release: rel}, nil
		}
		return nil, errors.Errorf("the release named %q is already deleted", name)
	}

	u.cfg.Log("uninstall: Deleting %s", name)
	rel.Info.Status = release.StatusUninstalling
	rel.Info.Deleted = helmtime.Now()
	rel.Info.Description = "Deletion in progress (or silently failed)"
	res := &release.UninstallReleaseResponse{Release: rel}

	if !u.DisableHooks {
		if err := u.cfg.execHook(rel, release.HookPreDelete, u.Timeout); err != nil {
			return res, err
		}
	} else {
		u.cfg.Log("delete hooks disabled for %s", name)
	}

	// From here on out, the release is currently considered to be in StatusUninstalling
	// state.
	if err := u.cfg.Releases.Update(rel); err != nil {
		u.cfg.Log("uninstall: Failed to store updated release: %s", err)
	}

	deletedResources, kept, errs := u.deleteRelease(rel)

	if kept != "" {
		kept = "These resources were kept due to the resource policy:\n" + kept
	}
	res.Info = kept

	if u.Wait {
		if kubeClient, ok := u.cfg.KubeClient.(kube.InterfaceExt); ok {
			if err := kubeClient.WaitForDelete(deletedResources, u.Timeout); err != nil {
				errs = append(errs, err)
			}
		}
	}

	if !u.DisableHooks {
		if err := u.cfg.execHook(rel, release.HookPostDelete, u.Timeout); err != nil {
			errs = append(errs, err)
		}
	}

	rel.Info.Status = release.StatusUninstalled
	if len(u.Description) > 0 {
		rel.Info.Description = u.Description
	} else {
		rel.Info.Description = "Uninstallation complete"
	}

	if !u.KeepHistory {
		u.cfg.Log("purge requested for %s", name)
		err := u.purgeReleases(rels...)
		if err != nil {
			errs = append(errs, errors.Wrap(err, "uninstall: Failed to purge the release"))
		}

		// Return the errors that occurred while deleting the release, if any
		if len(errs) > 0 {
			return res, errors.Errorf("uninstallation completed with %d error(s): %s", len(errs), joinErrors(errs))
		}

		return res, nil
	}

	if err := u.cfg.Releases.Update(rel); err != nil {
		u.cfg.Log("uninstall: Failed to store updated release: %s", err)
	}

	if len(errs) > 0 {
		return res, errors.Errorf("uninstallation completed with %d error(s): %s", len(errs), joinErrors(errs))
	}
	return res, nil
}
```
