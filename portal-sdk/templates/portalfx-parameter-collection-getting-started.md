<properties title="" pageTitle="Getting Started with Parameter Collection" description="Parameter Collection v3" authors="nickharris" />

{"gitdown": "include-file", "file": "./includes/portalfx-parameter-collection-related.md"}

## Getting Started with Parameter Collection V3
Implementing a parameter collection flow requires two distinct components to be implemented:

- Parameter Collector: responsibile for opening a child blade, supplying initial data to it, and may later receive back completed data.
- Parameter Provider:  responsible for recieving parameters from its parent blade, then later, when the user clicks on an action bar, supplies a result object to the parent blade and closes

## Implementing a ParameterCollector
To define a Parameter Collector the 'ParameterCollector' attribute is applied in PDL to the BladeAction that is responsible for launching your parameter provider blade

```xml
<Lens Name="PartLens">
  <Part PartKind="Button"
        Name="ParameterCollectorButton"
        ViewModel="{ViewModel Name=CollectorButtonViewModel,
        	Module=./CollectorAsButtonPart/ViewModels/CollectorButtonViewModel}" >

    <BladeAction Blade="ParameterProviderFormBlade"
    			 ParameterCollector="serverConfigCollector" />

  </Part>
</Lens>
```

Here the ParameterCollector property indicates which model property is responsible for sending initial data and receiving back a completed result. This attribute may only be used when the target blade contains a part marked with the 'ParameterProvider' attribute. Note that you can also specify additional BladeInput parameters if required.

Continuing with this example serverConfigCollector now needs to be defined as a property of type ParameterCollector<TResult> on CollectorButtonViewModel. So that the ParameterCollector and ParameterProvider can exchange data the TResult generic type must match the TResult generic type used on the parameter provider model.  In this scenario we're define a simple model called ServerConfig and declare our property:


```ts
export class CollectorButtonViewModel extends MsPortalFx.ViewModels.ButtonPart {

    public serverConfigCollector: MsPortalFx.ViewModels.ParameterCollector<ProviderModels.ServerConfig>;

    constructor(container: MsPortalFx.ViewModels.PartContainerContract,
    			initialState: any,
    			dataContext: ParameterCollectionArea.DataContext) {
		...
    }
}

```

To send initial data to the provider and optionally receive a result back you can specify callbacks during initialization of the ParameterCollector.

```ts
export class CollectorButtonViewModel extends MsPortalFx.ViewModels.ButtonPart {

    public serverConfigCollector: MsPortalFx.ViewModels.ParameterCollector<ProviderModels.ServerConfig>;

    constructor(container: MsPortalFx.ViewModels.PartContainerContract,
    			initialState: any,
    			dataContext: ParameterCollectionArea.DataContext) {

        super();

        this.serverConfigCollector = new MsPortalFx.ViewModels.ParameterCollector<ProviderModels.ServerConfig>(container, {
            supplyInitialData: () => {
                return {
                    diskSpaceBytes: ko.observable(108 * 1024 * 1024 * 1024),
                    serverName: ko.observable(ClientResources.parameterProviderDefaultServerName)
                };
            },
            receiveResult: (result: ProviderModels.ServerConfig) => {
                // A realistic parameter collector command would commence some operation on
                // receiving data. For this sample we have nothing particular to commence,
                // so we just show a notification to demonstrate that data was received.
                MsPortalFx.UI.NotificationManager
                    .create(ExtensionDefinition.NotificationDefinitions.ParameterCollectorReceivedResultNotification.name)
                    .raise(ExtensionDefinition.NotificationDefinitions.ParameterCollectorReceivedResultNotification.succeeded, null, ko.toJS(result));
            }
        });

        this.title(ClientResources.openProviderBlade);
    }
}
```

The complete list of supported options that can be supplied to the ParameterCollector constructor are defined are defined with the following signatures:

- **supplyInitialData? (): TResult**

    A callback that supplies initial data for the parameter provider in the child blade each time it opens.

    Note that the object received by the parameter provider will be a
    deep clone of the value you give, rather than the original instance,
    because it is passed (and sometimes stored) in a serialized form.
    @return Initial data for the child blade.

- **receiveResult? (result: TResult): void**

	A callback to be invoked when the child blade supplies a result and closes.
	@param result The result given by the child blade.


- **selectable?: Selectable<any>**

	The selectable associated with the same <BladeAction> as this parameter collector. The parameter collector will supply initial data to the child blade when this becomes selected.

	If not specified, this defaults to container.selectable (so it works with selectable parts without configuration).


- **supplyProviderConfig? (): any**


	A callback that supplies additional configuration options for the provider each time it opens. You can use this to pass non-editable data, for example configuring how a form will be displayed. @return Arbitrary configuration options for the child blade.


- **FormFieldValueAccessor?: FormFieldValueAccessors<TResult>**


	Provides an easy way to integrate a parameter collector with an EditScope.

	The collector will supply initial data to the provider from this edit scope property, and will automatically insert the provider's output into this edit scope property. The net result is that your parameter collector will act as an editor for the specified edit scope property.

	If you specify this option, do not also specify either supplyInitialData or receiveResult.

The full source of this ParameterCollector implementation can be found within the SamplesExtension under SamplesExtension\Extension\Client\ParameterCollection\CollectorAsButtonPart



## Implementing a ParameterProvider

To define a Parameter Provider the 'ParameterProvider' attribute is applied in PDL to the Part within the blade that is the target in the BladeAction that previously had a 'ParameterCollector' property defined. Continuing with the scenario presented in the Parameter Collector section above defining the ParameterCollector would be done as so:

```xml
<Blade Name="ParameterProviderFormBlade" ViewModel="{...}">
	<Lens Name="ParameterProviderFormLens">
  		<CustomPart Name="ParameterProviderFormPart"
					ViewModel="..."
					Template="{Html Source='Templates\\ParameterProviderForm.html'}"
              		InitialSize="HeroWideFitHeight"
					ParameterProvider="true">
  		</CustomPart>
	</Lens>

	<ActionBar ActionBarKind="Create" />
</Blade>
```

Here the ParameterProvider property indicates that this part is a provider which will have a property on the view model named parameterProvider that will receiving initial data from the collector (when provided) and returning back a completed result. 

Continuing with this example parameterProvider now needs to be defined as a property of type ParameterProvider<TResult, TEditScope> on ParameterProviderFormPartViewModel. To ensure the ParameterProvider and ParameterCollector can exchange data the TResult generic type must match the TResult generic type used on the parameter collector model.  In this scenario we define a simple model called ServerConfig for our TResult and for TEditScope  we use a different model of type ServerFormData. Note for many scenarios TResult and TEditScope may infact be the same type, to demonstrate that they can be structurally different we use different types here.


```ts
export class ParameterProviderFormPartViewModel extends MsPortalFx.ViewModels.Forms.Form.ViewModel<ProviderModels.ServerConfig> {

    public parameterProvider: MsPortalFx.ViewModels.ParameterProvider<ProviderModels.ServerConfig, ProviderModels.ServerFormData>;

    constructor(container: MsPortalFx.ViewModels.PartContainerContract,
    			initialState: any,
    			dataContext: ParameterCollectionArea.DataContext) {
		...
    }
	...
}
```

When you initialize the ParameterProvider there are two mandatory callbacks that must be specified within the constructor options. These two callbacks are *mapIncomingDataForEditScope* and *mapOutgoingDataForCollector* which allow you to map to and from TResult <> TEditScope.  Continuing with our example the ParameterProvider would be initialized as follows:

```ts

export class ParameterProviderFormPartViewModel extends MsPortalFx.ViewModels.Forms.Form<ProviderModels.ServerConfig> {

    public parameterProvider: MsPortalFx.ViewModels.ParameterProvider<ProviderModels.ServerConfig, ProviderModels.ServerFormData>;

    constructor(container: MsPortalFx.ViewModels.PartContainerContract,
    			initialState: any,
    			dataContext: ParameterCollectionArea.DataContext) {

        super(container);

        this.parameterProvider = new MsPortalFx.ViewModels.ParameterProvider<ProviderModels.ServerConfig, ProviderModels.ServerFormData>(container, {
            mapIncomingDataForEditScope: (incoming: ProviderModels.ServerConfig): ProviderModels.ServerFormData => {
                // Collectors are not required to supply complete initial data, so
                // providers must always fill in anything that is missing with defaults.
                incoming = incoming || <ProviderModels.ServerConfig>{};
                incoming.serverName = incoming.serverName || ko.observable("");
                incoming.diskSpaceBytes = incoming.diskSpaceBytes || ko.observable(defaultDiskSpace);

                return {
                    serverIdentifier: incoming.serverName,
                    fixedStorageGigabytes: ko.observable(incoming.diskSpaceBytes() / bytesPerGigabyte)
                };
            },

            mapOutgoingDataForCollector: (outgoing: ProviderModels.ServerFormData): ProviderModels.ServerConfig => {
                return {
                    serverName: outgoing.serverIdentifier,
                    diskSpaceBytes: ko.observable(outgoing.fixedStorageGigabytes() * bytesPerGigabyte)
                };
            }
        });

        // Use the form to edit the edit scope set up by the provider
        this.editScope = this.parameterProvider.editScope;

        this.serverIdentifierTextBox = new MsPortalFx.ViewModels.Forms.TextBox(container, this, "serverIdentifier");
        this.fixedStorageSlider = new MsPortalFx.ViewModels.Forms.Slider(container, this, "fixedStorageGigabytes", { min: ko.observable(50), max: ko.observable(1000) });
    }

    public serverIdentifierTextBox: MsPortalFx.ViewModels.Forms.TextBox;
    public fixedStorageSlider: MsPortalFx.ViewModels.Forms.Slider;
}

```
Here the data supplied by the collector to this provider of type ServerConfig (TResult) is  mapped to type ServerFormData (TEditScope) for this forms edit scope using callback *mapIncomingDataForEditScope*.  When the provider blade is closed by the user clicking Ok on the Create ActionBar *mapOutgoingDataForCollector* maps the data from the form of type ServerFormData (TEditScope) to ServerConfig(TResult) for the consuming collector which receives this data in its receiveResult callback.

For scenarios where you may want to invoke a provisioning operation that adds a startboard part and collapses the current journey when the provider is dismissed you can specify the *commitResult* callback.

The complete list of supported options that can be supplied to the ParameterProvider constructor are defined are defined with the following signatures:

- **mapIncomingDataForEditScope?(dataFromCollector: TResult): TEditScope**

	A callback that supplies the initial data for the provider's edit scope. This callback is mandatory, and must account for the possibility of the collector supplying incomplete information (or none at all) by returning the complete initial state for the edit scope.

	@param dataFromCollector The incoming initial data from the parameter collector, or null if the collector did not supply any.

	@return The desired initial edit scope data.

- **mapIncomingDataForEditScopeAsync?(dataFromCollector: TResult): Base.PromiseV<TEditScope>**

	An asynchronous callback that supplies the initial data for the provider's edit scope.
	This callback is mandatory, and must account for the possibility of the
    collector supplying incomplete information (or none at all) by returning
    the complete initial state for the edit scope.

    @param dataFromCollector The incoming initial data from the parameter collector, or null if the collector did not supply any.
    @return The desired initial edit scope data.

- **editScopeMetadataType?: string**

	The metadata type corresponding to the TResult generic parameter. This is used to configure the edit scope.


- **mapOutgoingDataForCollector(editScopeData: TEditScope): TResult**

    A mapping function that converts outgoing data from the provider's edit scope into the format you wish to return to the collector.

    Note that the object received by the parameter collector will be a deep clone of the value you give, rather than the original instance, because it is passed in a serialized form.

    @param editScopeData The data currently held by the parameter provider.
    @return The data that should be returned to the calling parameter collector.


- **commitResult? (editScopeData: TEditScope): void**

	A callback invoked when the user dismisses the provider. If you need to begin a provisioning operation that adds a startboard part and collapses the current journey, you can do so in this callback. You should not commence any other server-side operation from this callback, because the blade will have closed before it completes, so the user would not be able to see the result.

	@param editScopeData The data stored in the provider's edit scope.


The full source of this ParameterProvider implementation can be found within the SamplesExtension under SamplesExtension\Extension\Client\ParameterCollection\ParameterProviders.

Related Documentation: [ARM Provisioning API] (/documentation/articles/portalfx-provisioning-arm)
