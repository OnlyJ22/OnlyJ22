What part of the code allows for the web part to create lists needed on sharepoint 


Worked for 1m 47s
The list creation happens in the setup/provisioning path, not directly in the web part render code.

The main entry point is ConfigurationWizard.tsx. In finalizeSetup(), it does:

const provisioner = new ElementProvisioner();
await provisioner.ensureElements(configuration.schema);
That call goes into ElementProvisioner.ts, which is the class that actually provisions SharePoint artifacts. The specific line that creates a SharePoint list is here:

ElementProvisioner.ts
web.lists.ensure(title, description, template, false, listSettings);
That ensure(...) call is what creates the list if it does not already exist.

The lists it provisions are defined in the schema here:

RhythmOfBusinessCalendarSchema.ts
That schema includes:

ConfigurationList
EventsList
RefinersList
RefinerValuesList
ApproversList
ExternalListsConfigList
Each list’s fields/views are defined in files under src/schema/lists, for example EventsList.ts.

Also, when there is an upgrade scenario, the upgrade UI can provision lists through the same mechanism via Upgrade.tsx and CreateListUpgradeAction.ts.

If you want, I can map the exact first-load call chain from the web part entry file down to web.lists.ensure(...).


Does it create new sharepoint lists IF I manually created them myself?

No. It should not create duplicate lists if the lists already exist with the expected titles.

The key behavior is in ElementProvisioner.ts, which uses:

web.lists.ensure(title, description, template, false, listSettings);
ensure(...) means “get the list if it exists, otherwise create it.” So if you manually created a list with the same title, this code will reuse that list instead of creating another one.

There are two important caveats:

It matches by list title, so your manually created list would need the same title as the schema expects, such as those defined in RhythmOfBusinessCalendarSchema.ts.
On first creation, the code only runs the full field/view configuration when ensure(...) reports the list was newly created. You can see that in ElementProvisioner.ts, where it only calls _configureList(...) if listEnsureResult.created is true.
So the practical answer is:

If you manually created the lists first, it will usually not create new ones.
But it also may not fully add the expected columns/views/permissions to those existing manual lists during initial setup.
If you want, I can check whether your manual lists would still be compatible with this sample’s expected schema.


if there's exisiting ones does it delete the records?

No. Existing list items are not deleted during setup just because the list already exists.

The provisioning path only creates the list when missing, via ElementProvisioner.ts. If the list already exists, it does not call the new-list configuration branch at ElementProvisioner.ts.

I also did not find any setup logic that clears list items on existing lists. The only place that deletes lists/items is a developer utility in OnlineConfigurationService.ts, schema.obliterate(), which:

deletes the active configuration item
deletes the schema lists
That is not normal setup behavior.

So the short answer is: existing records are not deleted by the normal provisioning/setup flow.


so this line of code prevents another list from being created if the title matches?

web.lists.ensure(title, description, template, false, listSettings);

Yes.

ElementProvisioner.ts calls:

web.lists.ensure(title, description, template, false, listSettings);
That means:

if a SharePoint list with that title already exists, it returns the existing list
if it does not exist, it creates it
So this is the line that prevents a second list from being created when the title already matches.

One important nuance: it prevents duplicate creation, but it does not guarantee an existing manually-created list has the right schema. In this codebase, extra configuration only happens when ensure(...).created is true, at ElementProvisioner.ts.


If I install one version of this webpart with its own GUID and the second one with its own GUID, lets say the first one creates the lists and then the second is added they are seperate webparts that do truly use the same lists but would it being seperate allow it to create its own?


No. A different web part GUID by itself would not make it create separate SharePoint lists.

What matters here is the schema and list titles, not the web part component ID. This sample provisions lists by title through ElementProvisioner.ts:

web.lists.ensure(title, description, template, false, listSettings);
So if:

web part A and web part B run on the same SharePoint site
both use the same schema/list titles
both call the same provisioning code
then the second one will find the existing lists and reuse them. It will not create another set just because its SPFx manifest GUID is different.

It would only create its own separate lists if at least one of these changes:

the list titles are different
the provisioning code/schema is changed to point to different list definitions
it is installed on a different SharePoint site/web
the code is modified to namespace list names per web part instance or per manifest ID
In this project, the list names come from the schema definitions like RhythmOfBusinessCalendarSchema.ts and the individual list files under src/schema/lists. The web part GUID is not used there to generate list names.

So in your scenario, two separately packaged versions with different web part GUIDs would still target the same lists if they are deployed to the same site and keep the same schema/list titles.

does it change the permissions?


Yes, but only when the list is newly created by this provisioning flow.

The permission logic is in ElementProvisioner.ts, where _configureList(...) calls _configurePermissions(...). That configuration includes breaking inheritance and assigning roles, shown in ElementProvisioner.ts.

The important limitation is earlier in the same file:

ElementProvisioner.ts uses web.lists.ensure(...)
ElementProvisioner.ts only calls _configureList(...) if listEnsureResult.created is true
So:

if the list does not exist and this code creates it, yes, it applies the permissions defined in the schema
if the list already exists, this normal setup path does not reapply or change permissions
Example: the events list defines its intended permissions in EventsList.ts, but those would only be enforced during creation unless some other upgrade/admin path explicitly runs permission configuration later.

So the practical answer is: it can change permissions, but not on an already existing list during normal setup.
