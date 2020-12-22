### k8s 其中一个节点发生网络隔离导致 kubectl get 命令变慢分析

staging/src/k8s.io/kubectl/pkg/cmd/get/get.go：

1. 入口：

   ```go
   	cmd := &cobra.Command{
   		Use:                   "get [(-o|--output=)json|yaml|wide|custom-columns=...|custom-columns-file=...|go-template=...|go-template-file=...|jsonpath=...|jsonpath-file=...] (TYPE[.VERSION][.GROUP] [NAME | -l label] | TYPE[.VERSION][.GROUP]/NAME ...) [flags]",
   		DisableFlagsInUseLine: true,
   		Short:                 i18n.T("Display one or many resources"),
   		Long:                  getLong + "\n\n" + cmdutil.SuggestAPIResources(parent),
   		Example:               getExample,
   		Run: func(cmd *cobra.Command, args []string) {
   			cmdutil.CheckErr(o.Complete(f, cmd, args))
   			cmdutil.CheckErr(o.Validate(cmd))
   			cmdutil.CheckErr(o.Run(f, cmd, args))
   		},
   		SuggestFor: []string{"list", "ps"},
   	}
   ```

2. o.Run(f, cmd, args) 方法逻辑：

   ```go
   	r := f.NewBuilder().
   		Unstructured().
   		NamespaceParam(o.Namespace).DefaultNamespace().AllNamespaces(o.AllNamespaces).
   		FilenameParam(o.ExplicitNamespace, &o.FilenameOptions).
   		LabelSelectorParam(o.LabelSelector).
   		FieldSelectorParam(o.FieldSelector).
   		RequestChunksOf(chunkSize).
   		ResourceTypeOrNameArgs(true, args...).
   		ContinueOnError().
   		Latest().
   		Flatten().
   		TransformRequests(o.transformRequests).
   		Do()
   ```

3. f.NewBuilder()

   ```
   func NewBuilder(restClientGetter RESTClientGetter) *Builder {
   	categoryExpanderFn := func() (restmapper.CategoryExpander, error) {
   		discoveryClient, err := restClientGetter.ToDiscoveryClient()
   		if err != nil {
   			return nil, err
   		}
   		return restmapper.NewDiscoveryCategoryExpander(discoveryClient), err
   	}
   
   	return newBuilder(
   		restClientGetter.ToRESTConfig,
   		(&cachingRESTMapperFunc{delegate: restClientGetter.ToRESTMapper}).ToRESTMapper,
   		(&cachingCategoryExpanderFunc{delegate: categoryExpanderFn}).ToCategoryExpander,
   	)
   }
   ```

   restClientGetter 通过传入 cmd 的 f.clientGetter 获得。f 的 构造方法如下：

   /vendor/k8s.io/kubectl/pkg/cmd/util/factory_client_access.go：

   ```go
   kubeConfigFlags := genericclioptions.NewConfigFlags(true).WithDeprecatedPasswordFlag()
   kubeConfigFlags.AddFlags(flags)
   matchVersionKubeConfigFlags := cmdutil.NewMatchVersionFlags(kubeConfigFlags)
   matchVersionKubeConfigFlags.AddFlags(cmds.PersistentFlags())
   
   f := cmdutil.NewFactory(matchVersionKubeConfigFlags)
   
   func NewFactory(clientGetter genericclioptions.RESTClientGetter) Factory {
   	if clientGetter == nil {
   		panic("attempt to instantiate client_access_factory with nil clientGetter")
   	}
   
   	f := &factoryImpl{
   		clientGetter: clientGetter,
   	}
   
   	return f
   }
   ```

   matchVersionKubeConfigFlags 代理了 kubeConfigFlags。kubeConfigFlags 本质是一个 *ConfigFlags，它实现了 RESTClientGetter 接口。

4. f.Unstructured() 方法：

   ```go
   func (b *Builder) Unstructured() *Builder {
   	if b.mapper != nil {
   		b.errs = append(b.errs, fmt.Errorf("another mapper was already selected, cannot use unstructured types"))
   		return b
   	}
   	b.objectTyper = unstructuredscheme.NewUnstructuredObjectTyper()
   	b.mapper = &mapper{
   		localFn:      b.isLocal,
   		restMapperFn: b.restMapperFn,
   		clientFn:     b.getClient,
   		decoder:      &metadataValidatingDecoder{unstructured.UnstructuredJSONScheme},
   	}
   
   	return b
   }
   ```

   b.mapper 的 restMapperFn 成员通过通过 newBuilder() 方法已经传入。

5. f.ResourceTypeOrNameArgs(true, args...) 方法：

   ```go
   func (b *Builder) ResourceTypeOrNameArgs(allowEmptySelector bool, args ...string) *Builder {
   	args = normalizeMultipleResourcesArgs(args)
     // 省略代码
   	if len(args) > 0 {
   		// Try replacing aliases only in types
   		args[0] = b.ReplaceAliases(args[0])
   	}
   	// 省略代码
   	return b
   }
   ```

6. b.ReplaceAliases(args[0]) 方法：

   ```go
   func (b *Builder) ReplaceAliases(input string) string {
   	replaced := []string{}
   	for _, arg := range strings.Split(input, ",") {
   		if b.categoryExpanderFn == nil {
   			continue
   		}
   		categoryExpander, err := b.categoryExpanderFn()
   		if err != nil {
   			b.AddError(err)
   			continue
   		}
   
   		if resources, ok := categoryExpander.Expand(arg); ok {
   			asStrings := []string{}
   			for _, resource := range resources {
   				if len(resource.Group) == 0 {
   					asStrings = append(asStrings, resource.Resource)
   					continue
   				}
   				asStrings = append(asStrings, resource.Resource+"."+resource.Group)
   			}
   			arg = strings.Join(asStrings, ",")
   		}
   		replaced = append(replaced, arg)
   	}
   	return strings.Join(replaced, ",")
   }
   ```

   b.categoryExpanderFn() 返回的 categoryExpander 本质是一个 discoveryCategoryExpander，它代理了一个 discoveryClient。

7. categoryExpander.Expand(arg) 方法：

   ```go
   func (e discoveryCategoryExpander) Expand(category string) ([]schema.GroupResource, bool) {
   	// Get all supported resources for groups and versions from server, if no resource found, fallback anyway.
   	_, apiResourceLists, _ := e.discoveryClient.ServerGroupsAndResources()
     // 省略代码
     ret, ok := discoveredExpansions[category]
   	return ret, ok
   }
   ```

8. e.discoveryClient.ServerGroupsAndResources()：

   ```
   // ServerGroupsAndResources returns the supported groups and resources for all groups and versions.
   func (d *CachedDiscoveryClient) ServerGroupsAndResources() ([]*metav1.APIGroup, []*metav1.APIResourceList, error) {
   	return discovery.ServerGroupsAndResources(d)
   }
   ```

9. discovery.ServerGroupsAndResources(d)：

   ```
   func ServerGroupsAndResources(d DiscoveryInterface) ([]*metav1.APIGroup, []*metav1.APIResourceList, error) {
   	sgs, err := d.ServerGroups()
   
     // 省略代码
   
   	return resultGroups, result, &ErrGroupDiscoveryFailed{Groups: failedGroups}
   }
   ```

10. d.ServerGroups() 方法：

    ```go
    func (d *CachedDiscoveryClient) ServerGroups() (*metav1.APIGroupList, error) {
    	filename := filepath.Join(d.cacheDirectory, "servergroups.json")
    	cachedBytes, err := d.getCachedFile(filename)
    	// don't fail on errors, we either don't have a file or won't be able to run the cached check. Either way we can fallback.
    	if err == nil {
    		cachedGroups := &metav1.APIGroupList{}
    		if err := runtime.DecodeInto(scheme.Codecs.UniversalDecoder(), cachedBytes, cachedGroups); err == nil {
    			klog.V(10).Infof("returning cached discovery info from %v", filename)
    			return cachedGroups, nil
    		}
    	}
    
    	liveGroups, err := d.delegate.ServerGroups()
    	if err != nil {
    		klog.V(3).Infof("skipped caching discovery info due to %v", err)
    		return liveGroups, err
    	}
    	if liveGroups == nil || len(liveGroups.Groups) == 0 {
    		klog.V(3).Infof("skipped caching discovery info, no groups found")
    		return liveGroups, err
    	}
    
    	if err := d.writeCachedFile(filename, liveGroups); err != nil {
    		klog.V(1).Infof("failed to write cache to %v due to %v", filename, err)
    	}
    
    	return liveGroups, nil
    }
    ```

    该方法会去读取 ~/.kube/cache/discovery/servergroups.json，如果该文件过期，则会重新缓存该文件。

11. d.delegate.ServerGroups() 获取缓存：

    ```
    func (d *DiscoveryClient) ServerGroups() (apiGroupList *metav1.APIGroupList, err error) {
    	// Get the groupVersions exposed at /api
    	v := &metav1.APIVersions{}
    	err = d.restClient.Get().AbsPath(d.LegacyPrefix).Do(context.TODO()).Into(v)
    	apiGroup := metav1.APIGroup{}
    	if err == nil && len(v.Versions) != 0 {
    		apiGroup = apiVersionsToAPIGroup(v)
    	}
    	if err != nil && !errors.IsNotFound(err) && !errors.IsForbidden(err) {
    		return nil, err
    	}
    
    	// Get the groupVersions exposed at /apis
    	apiGroupList = &metav1.APIGroupList{}
    	err = d.restClient.Get().AbsPath("/apis").Do(context.TODO()).Into(apiGroupList)
    	if err != nil && !errors.IsNotFound(err) && !errors.IsForbidden(err) {
    		return nil, err
    	}
    	// to be compatible with a v1.0 server, if it's a 403 or 404, ignore and return whatever we got from /api
    	if err != nil && (errors.IsNotFound(err) || errors.IsForbidden(err)) {
    		apiGroupList = &metav1.APIGroupList{}
    	}
    
    	// prepend the group retrieved from /api to the list if not empty
    	if len(v.Versions) != 0 {
    		apiGroupList.Groups = append([]metav1.APIGroup{apiGroup}, apiGroupList.Groups...)
    	}
    	return apiGroupList, nil
    }
    ```

    在缓存 *apis/metrics*.*k8s*.io 和 apis/*custom*.*metrics*.*k8s*.io 时，由于网络隔离的原因，导致这两个接口所要访问的 metrics-server 需要重新调度，所以执行 kubectl get 命令变慢。

12. f.Do() 方法返回具体的资源。