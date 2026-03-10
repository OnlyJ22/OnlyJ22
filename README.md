so its this then?

    private async _configurePermissions(listDefinition: IListDefinition, web: IWeb, forcePermissionReset: boolean): Promise<void> {
        const { permissions } = listDefinition;

        const eh = new ErrorHandler();
        const batch = web.createBatch();
        const list = web.lists.getByTitle(listDefinition.title).inBatch(batch);

        if (forcePermissionReset) {
            list.resetRoleInheritance().catch(eh.catch);
        }

        if (permissions) {
            const [
                roles,
                adminRoleDefinition,
                currentUser
            ] = await Promise.all([
                Promise.all(this._resolveUserRoles(web, permissions)),
                web.roleDefinitions.getByType(RoleType.Administrator)(),
                web.currentUser()
            ]);

            list.breakRoleInheritance(permissions.copyRoleAssignments, false).catch(eh.catch);

            roles.filter(role => !!role.roleDefinitionId).forEach(role => {
                const assignments = list.roleAssignments.inBatch(batch);

                switch (role.userRole.operation) {
                    case RoleOperation.Add:
                        assignments.add(role.principalId, role.roleDefinitionId).catch(eh.catch);
                        break;
                    case RoleOperation.Remove:
                        assignments.remove(role.principalId, role.roleDefinitionId).catch(eh.catch);
                        break;
                }
            });

            list.roleAssignments.inBatch(batch).remove(currentUser.Id, adminRoleDefinition.Id).catch(eh.catch);
        }

        await batch.execute();
        eh.throwIfError();
    }


Yes.

That method is the core of the problem.

Why:

list.breakRoleInheritance(permissions.copyRoleAssignments, false)
your schemas use copyRoleAssignments: false
so the new list does not keep the subsite’s existing direct user permissions
Then this code adds permissions only for the associated groups resolved by _resolveUserRoles(...):

ownerGroup
memberGroup
visitorGroup
And after that it does this:

list.roleAssignments.inBatch(batch).remove(currentUser.Id, adminRoleDefinition.Id)
So if you were not in one of those associated groups, your effective access could disappear during provisioning.

In plain English:

yes, this function is the reason
especially the combination of breakRoleInheritance(false, false) plus only re-adding group-based permissions
The most important line is:

list.breakRoleInheritance(permissions.copyRoleAssignments, false)
because with copyRoleAssignments: false, direct named-user access is dropped.

And this line can make it worse:

list.roleAssignments.inBatch(batch).remove(currentUser.Id, adminRoleDefinition.Id)
because it explicitly removes the current user’s admin role on the list.

So your reading is correct: this is the code path causing the issue.

how would i fix this?


The safest fix is to preserve inherited permissions when the list is created.

Change the list schemas from copyRoleAssignments: false to copyRoleAssignments: true.

Example in ConfigurationList.ts (line 97) and the other list files:

permissions: {
    copyRoleAssignments: true,
    userRoles: [
        { operation: RoleOperation.Add, roleType: RoleType.Administrator, userType: 'ownerGroup' },
        { operation: RoleOperation.Add, roleType: RoleType.Reader, userType: 'memberGroup' },
        { operation: RoleOperation.Add, roleType: RoleType.Reader, userType: 'visitorGroup' }
    ]
}
Why this works:

the list keeps existing inherited/direct permissions from the subsite
your direct named-user access survives
the app can still add its Owners/Members/Visitors group assignments
I would also remove or guard this line in ElementProvisioner.ts (line 329):

list.roleAssignments.inBatch(batch).remove(currentUser.Id, adminRoleDefinition.Id).catch(eh.catch);
Safer options:

simplest: delete that line
safer conditional: only remove the current user if they were explicitly added by this provisioning logic
Best practical fix:

Set all schema lists to copyRoleAssignments: true
Remove the remove(currentUser.Id, adminRoleDefinition.Id) line
That gives the least surprising behavior and stops setup from locking out directly-permissioned users.

If you want, I can patch all of those files for you now.
