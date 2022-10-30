---
title: k8s源码分析：kube-apiserver –admission-control及–enable-admission-plugins –disable-admission-plugins参数差异
author: 阿辉
date: 2021-02-01T10:26:49+00:00
categories:
- Kubernetes
- ApiServer
tags:
- Kubernetes
- ApiServer
keywords:
- Kubernetes
- ApiServer
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---

kubernetes源码分支：1.18

先说结论，kube-apiserver启动时：

- --admission-control参数带的插件将是apiserver启动的插件，不包括默认插件
- --admission-control和--enable-admission-plugins --disable-admission-plugins不能同时使用
- --enable-admission-plugins参数不需要按加载顺序填写
- 不使用--admission-control参数时，api server会同时启动默认插件
- --enable-admission-plugins参数启用时，api server会同时启动默认插件，除非使用了--disable-admission-plugins显示的关闭某个插件
- --enable-admission-plugins和--disable-admission-plugins如果同时填写了某一个插件，这个插件将会被加载


<!--more-->

源码pkg/kubeapiserver/options/plugins.go内按加载顺序定义了所有的插件,同时也定义了默认的插件：

```go
// 所有插件（带加载顺序）
var AllOrderedPlugins = []string{
	admit.PluginName,                        // AlwaysAdmit
	autoprovision.PluginName,                // NamespaceAutoProvision
	lifecycle.PluginName,                    // NamespaceLifecycle
	exists.PluginName,                       // NamespaceExists
	scdeny.PluginName,                       // SecurityContextDeny
	antiaffinity.PluginName,                 // LimitPodHardAntiAffinityTopology
	podpreset.PluginName,                    // PodPreset
	limitranger.PluginName,                  // LimitRanger
	serviceaccount.PluginName,               // ServiceAccount
	noderestriction.PluginName,              // NodeRestriction
	nodetaint.PluginName,                    // TaintNodesByCondition
	alwayspullimages.PluginName,             // AlwaysPullImages
	imagepolicy.PluginName,                  // ImagePolicyWebhook
	podsecuritypolicy.PluginName,            // PodSecurityPolicy
	podnodeselector.PluginName,              // PodNodeSelector
	podpriority.PluginName,                  // Priority
	defaulttolerationseconds.PluginName,     // DefaultTolerationSeconds
	podtolerationrestriction.PluginName,     // PodTolerationRestriction
	exec.DenyEscalatingExec,                 // DenyEscalatingExec
	exec.DenyExecOnPrivileged,               // DenyExecOnPrivileged
	eventratelimit.PluginName,               // EventRateLimit
	extendedresourcetoleration.PluginName,   // ExtendedResourceToleration
	label.PluginName,                        // PersistentVolumeLabel
	setdefault.PluginName,                   // DefaultStorageClass
	storageobjectinuseprotection.PluginName, // StorageObjectInUseProtection
	gc.PluginName,                           // OwnerReferencesPermissionEnforcement
	resize.PluginName,                       // PersistentVolumeClaimResize
	runtimeclass.PluginName,                 // RuntimeClass
	certapproval.PluginName,                 // CertificateApproval
	certsigning.PluginName,                  // CertificateSigning
	certsubjectrestriction.PluginName,       // CertificateSubjectRestriction
	defaultingressclass.PluginName,          // DefaultIngressClass

	// new admission plugins should generally be inserted above here
	// webhook, resourcequota, and deny plugins must go at the end

	mutatingwebhook.PluginName,   // MutatingAdmissionWebhook
	validatingwebhook.PluginName, // ValidatingAdmissionWebhook
	resourcequota.PluginName,     // ResourceQuota
	deny.PluginName,              // AlwaysDeny
}

// RegisterAllAdmissionPlugins registers all admission plugins and
// sets the recommended plugins order.
func RegisterAllAdmissionPlugins(plugins *admission.Plugins) {
	admit.Register(plugins) // DEPRECATED as no real meaning
	alwayspullimages.Register(plugins)
	antiaffinity.Register(plugins)
	defaulttolerationseconds.Register(plugins)
	defaultingressclass.Register(plugins)
	deny.Register(plugins) // DEPRECATED as no real meaning
	eventratelimit.Register(plugins)
	exec.Register(plugins)
	extendedresourcetoleration.Register(plugins)
	gc.Register(plugins)
	imagepolicy.Register(plugins)
	limitranger.Register(plugins)
	autoprovision.Register(plugins)
	exists.Register(plugins)
	noderestriction.Register(plugins)
	nodetaint.Register(plugins)
	label.Register(plugins) // DEPRECATED, future PVs should not rely on labels for zone topology
	podnodeselector.Register(plugins)
	podpreset.Register(plugins)
	podtolerationrestriction.Register(plugins)
	runtimeclass.Register(plugins)
	resourcequota.Register(plugins)
	podsecuritypolicy.Register(plugins)
	podpriority.Register(plugins)
	scdeny.Register(plugins)
	serviceaccount.Register(plugins)
	setdefault.Register(plugins)
	resize.Register(plugins)
	storageobjectinuseprotection.Register(plugins)
	certapproval.Register(plugins)
	certsigning.Register(plugins)
	certsubjectrestriction.Register(plugins)
}

// DefaultOffAdmissionPlugins get admission plugins off by default for kube-apiserver.
func DefaultOffAdmissionPlugins() sets.String {
    //默认开启的插件
	defaultOnPlugins := sets.NewString(
		lifecycle.PluginName,                    //NamespaceLifecycle
		limitranger.PluginName,                  //LimitRanger
		serviceaccount.PluginName,               //ServiceAccount
		setdefault.PluginName,                   //DefaultStorageClass
		resize.PluginName,                       //PersistentVolumeClaimResize
		defaulttolerationseconds.PluginName,     //DefaultTolerationSeconds
		mutatingwebhook.PluginName,              //MutatingAdmissionWebhook
		validatingwebhook.PluginName,            //ValidatingAdmissionWebhook
		resourcequota.PluginName,                //ResourceQuota
		storageobjectinuseprotection.PluginName, //StorageObjectInUseProtection
		podpriority.PluginName,                  //PodPriority
		nodetaint.PluginName,                    //TaintNodesByCondition
		runtimeclass.PluginName,                 //RuntimeClass, gates internally on the feature
		certapproval.PluginName,                 // CertificateApproval
		certsigning.PluginName,                  // CertificateSigning
		certsubjectrestriction.PluginName,       // CertificateSubjectRestriction
		defaultingressclass.PluginName,          //DefaultIngressClass
	)

	return sets.NewString(AllOrderedPlugins...).Difference(defaultOnPlugins)
}
```

源码pkg/kubeapiserver/options/admission.go:

```go
// AddFlags adds flags related to admission for kube-apiserver to the specified FlagSet
func (a *AdmissionOptions) AddFlags(fs *pflag.FlagSet) {
	fs.StringSliceVar(&a.PluginNames, "admission-control", a.PluginNames, ""+
		"Admission is divided into two phases. "+
		"In the first phase, only mutating admission plugins run. "+
		"In the second phase, only validating admission plugins run. "+
		"The names in the below list may represent a validating plugin, a mutating plugin, or both. "+
		"The order of plugins in which they are passed to this flag does not matter. "+
		"Comma-delimited list of: "+strings.Join(a.GenericAdmission.Plugins.Registered(), ", ")+".")
	fs.MarkDeprecated("admission-control", "Use --enable-admission-plugins or --disable-admission-plugins instead. Will be removed in a future version.")
	fs.Lookup("admission-control").Hidden = false

	a.GenericAdmission.AddFlags(fs)
}

// ApplyTo adds the admission chain to the server configuration.
// Kube-apiserver just call generic AdmissionOptions.ApplyTo.
func (a *AdmissionOptions) ApplyTo(
	c *server.Config,
	informers informers.SharedInformerFactory,
	kubeAPIServerClientConfig *rest.Config,
	features featuregate.FeatureGate,
	pluginInitializers ...admission.PluginInitializer,
) error {
	if a == nil {
		return nil
	}

	if a.PluginNames != nil {
		// pass PluginNames to generic AdmissionOptions
		a.GenericAdmission.EnablePlugins, a.GenericAdmission.DisablePlugins = computePluginNames(a.PluginNames, a.GenericAdmission.RecommendedPluginOrder)
	}

	return a.GenericAdmission.ApplyTo(c, informers, kubeAPIServerClientConfig, features, pluginInitializers...)
}

// explicitly disable all plugins that are not in the enabled list
func computePluginNames(explicitlyEnabled []string, all []string) (enabled []string, disabled []string) {
	return explicitlyEnabled, sets.NewString(all...).Difference(sets.NewString(explicitlyEnabled...)).List()
}

```

- AddFlags函数内，admission-control启动参数带的值存在了a.PluginNames内

- ApplyTo函数内，31行和39行，将a.PluginNames内的插件直接传给了a.GenericAdmission.EnablePlugins，同时与a.GenericAdmission.RecommendedPluginOrder差异对比，差异的部分给了a.GenericAdmission.DisablePlugins

- --enable-admission-plugins --disable-admission-plugins参数的值分别存a.GenericAdmission.EnablePlugins和a.GenericAdmission.DisablePlugins内。

源码k8s.io/apiserver/pkg/server/options/admission.go
```go
// AdmissionOptions holds the admission options
type AdmissionOptions struct {
	// RecommendedPluginOrder holds an ordered list of plugin names we recommend to use by default
	RecommendedPluginOrder []string
	// DefaultOffPlugins is a set of plugin names that is disabled by default
	DefaultOffPlugins sets.String

	// EnablePlugins indicates plugins to be enabled passed through `--enable-admission-plugins`.
	EnablePlugins []string
	// DisablePlugins indicates plugins to be disabled passed through `--disable-admission-plugins`.
	DisablePlugins []string
	// ConfigFile is the file path with admission control configuration.
	ConfigFile string
	// Plugins contains all registered plugins.
	Plugins *admission.Plugins
	// Decorators is a list of admission decorator to wrap around the admission plugins
	Decorators admission.Decorators
}

// AddFlags adds flags related to admission for a specific APIServer to the specified FlagSet
func (a *AdmissionOptions) AddFlags(fs *pflag.FlagSet) {
	if a == nil {
		return
	}

	fs.StringSliceVar(&a.EnablePlugins, "enable-admission-plugins", a.EnablePlugins, ""+
		"admission plugins that should be enabled in addition to default enabled ones ("+
		strings.Join(a.defaultEnabledPluginNames(), ", ")+"). "+
		"Comma-delimited list of admission plugins: "+strings.Join(a.Plugins.Registered(), ", ")+". "+
		"The order of plugins in this flag does not matter.")
	fs.StringSliceVar(&a.DisablePlugins, "disable-admission-plugins", a.DisablePlugins, ""+
		"admission plugins that should be disabled although they are in the default enabled plugins list ("+
		strings.Join(a.defaultEnabledPluginNames(), ", ")+"). "+
		"Comma-delimited list of admission plugins: "+strings.Join(a.Plugins.Registered(), ", ")+". "+
		"The order of plugins in this flag does not matter.")
	fs.StringVar(&a.ConfigFile, "admission-control-config-file", a.ConfigFile,
		"File with admission control configuration.")
}

// ApplyTo adds the admission chain to the server configuration.
// In case admission plugin names were not provided by a cluster-admin they will be prepared from the recommended/default values.
// In addition the method lazily initializes a generic plugin that is appended to the list of pluginInitializers
// note this method uses:
//  genericconfig.Authorizer
func (a *AdmissionOptions) ApplyTo(
	c *server.Config,
	informers informers.SharedInformerFactory,
	kubeAPIServerClientConfig *rest.Config,
	features featuregate.FeatureGate,
	pluginInitializers ...admission.PluginInitializer,
) error {
	if a == nil {
		return nil
	}

	// Admission depends on CoreAPI to set SharedInformerFactory and ClientConfig.
	if informers == nil {
		return fmt.Errorf("admission depends on a Kubernetes core API shared informer, it cannot be nil")
	}

	pluginNames := a.enabledPluginNames()

	pluginsConfigProvider, err := admission.ReadAdmissionConfiguration(pluginNames, a.ConfigFile, configScheme)
	if err != nil {
		return fmt.Errorf("failed to read plugin config: %v", err)
	}

	clientset, err := kubernetes.NewForConfig(kubeAPIServerClientConfig)
	if err != nil {
		return err
	}
	genericInitializer := initializer.New(clientset, informers, c.Authorization.Authorizer, features)
	initializersChain := admission.PluginInitializers{}
	pluginInitializers = append(pluginInitializers, genericInitializer)
	initializersChain = append(initializersChain, pluginInitializers...)

	admissionChain, err := a.Plugins.NewFromPlugins(pluginNames, pluginsConfigProvider, initializersChain, a.Decorators)
	if err != nil {
		return err
	}

	c.AdmissionControl = admissionmetrics.WithStepMetrics(admissionChain)
	return nil
}

// enabledPluginNames makes use of RecommendedPluginOrder, DefaultOffPlugins,
// EnablePlugins, DisablePlugins fields
// to prepare a list of ordered plugin names that are enabled.
func (a *AdmissionOptions) enabledPluginNames() []string {
	allOffPlugins := append(a.DefaultOffPlugins.List(), a.DisablePlugins...)
	disabledPlugins := sets.NewString(allOffPlugins...)
	enabledPlugins := sets.NewString(a.EnablePlugins...)
	disabledPlugins = disabledPlugins.Difference(enabledPlugins)

	orderedPlugins := []string{}
	for _, plugin := range a.RecommendedPluginOrder {
		if !disabledPlugins.Has(plugin) {
			orderedPlugins = append(orderedPlugins, plugin)
		}
	}

	return orderedPlugins
}

//Return names of plugins which are enabled by default
func (a *AdmissionOptions) defaultEnabledPluginNames() []string {
	defaultOnPluginNames := []string{}
	for _, pluginName := range a.RecommendedPluginOrder {
		if !a.DefaultOffPlugins.Has(pluginName) {
			defaultOnPluginNames = append(defaultOnPluginNames, pluginName)
		}
	}

	return defaultOnPluginNames
}
```

- a.EnablePlugins用于存放--enable-admission-plugins参数的值
- a.DisablePlugins用于存放--disable-admission-plugins参数的值
- ApplyTo方法的关键在enabledPluginNames，enabledPluginNames确定了需要加载的插件列表。
- allOffPlugins为默认关闭的插件列表和--disable-admission-plugins参数带的插件列表
- 再使用allOffPlugins和enabledPlugins进行差异对比，差异的部分保存在disabledPlugins
- 最后按RecommendedPluginOrder(RecommendedPluginOrder为所有插件)内的顺序排序和过滤（过滤到disabledPlugins存在的插件）并返回需要加载的插件列表。

这样就说明：

- 如果--enable-admission-plugins和--disable-admission-plugins如果同时填写了某一个插件，这个插件将会被加载。
- 如果只是使用--enable-admission-plugins和--disable-admission-plugins参数，默认插件是会加载的

