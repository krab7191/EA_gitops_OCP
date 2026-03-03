# EEM User Groups Configuration Guide

## Problem Summary

Event Endpoint Management (EEM) was configured with OIDC authentication but **missing the `userGroupClaimPointer`** configuration. This caused:

- ❌ "User groups" menu not appearing in EEM UI
- ❌ Unable to delete Kafka clusters (403 Forbidden)
- ❌ Unable to assign cluster maintainers
- ✅ Could create clusters (requires `admin` role only)
- ✅ Could not delete clusters (requires cluster maintainer group membership)

## Root Cause

According to [IBM EEM Documentation](https://ibm.github.io/event-automation/eem/security/groups/):

> "Note: Access to user groups is available with OpenID Connect (OIDC) authentication only."

The `userGroupClaimPointer` parameter tells EEM where to find user groups in the OIDC token. Without it:
- User groups feature is completely disabled
- Cluster maintainers cannot be assigned
- Event source editors cannot be configured
- Option viewers cannot be managed

## Solution Applied

### Configuration Changes

**1. Updated `instances/event-endpoint-mgmt/eventendpointmgmt.yaml`:**

```yaml
authConfig:
  authType: OIDC
  oidcConfig:
    # For role-based access (admin, author, viewer)
    authorizationClaimPointer: /roles
    
    # For user groups (cluster maintainers, event source editors, option viewers)
    # THIS WAS MISSING - NOW ADDED
    userGroupClaimPointer: /groups
```

**2. Updated `instances/event-endpoint-mgmt/user-roles.yaml`:**

Changed admin role mapping to explicitly include all three roles:

```json
{
  "id": "admin",
  "roles": ["admin", "author", "viewer"]
}
```

**Why assign all three roles?**
- While admin hierarchically includes author and viewer permissions, explicitly assigning all three ensures maximum visibility across all UI features
- Guarantees access to "User groups" menu and all administrative functions
- Provides complete access to view, create, and manage all resources
- Eliminates any potential permission gaps in the UI

### How It Works

**authorizationClaimPointer** (`/roles`):
- Maps to `token.roles` array in Keycloak token
- Controls UI access levels: `admin`, `author`, `viewer`
- Required for basic EEM access
- Users with "admin" in their token get all three EEM roles

**userGroupClaimPointer** (`/groups`):
- Maps to `token.groups` array in Keycloak token
- Enables user groups feature in EEM
- Required for cluster maintainers, event source editors, option viewers
- EEM checks: ID token → Access token → User Info (in that order)

**additionalScopes** (`["offline_access"]`):
- Optional, only needed if using Admin API with user groups
- Allows EEM to refresh tokens and get updated group memberships
- Not all OIDC providers support this scope

## Next Steps

### 1. Apply the Configuration

```bash
# Commit and push changes
cd "/Users/karsten/OneDrive - IBM/Client Engineering/Engagements/FL/FSU_Event-Automation/EA_gitops_OCP"
git add instances/event-endpoint-mgmt/eventendpointmgmt.yaml \
        instances/event-endpoint-mgmt/user-roles.yaml \
        docs/EEM-USER-GROUPS-SETUP.md
git commit -m "fix: Add userGroupClaimPointer and comprehensive role assignments for EEM"
git push

# ArgoCD will sync automatically, or manually sync:
# argocd app sync event-endpoint-mgmt-app
```

### 2. Configure Keycloak Groups

In Keycloak admin console (FSU-EA realm):

```
1. Create groups:
   - Navigate to: Groups
   - Click "Create group"
   - Name: "cluster-admins" (or your preferred name)
   - Save

2. Add users to groups:
   - Navigate to: Users → testuser
   - Click "Groups" tab
   - Click "Join Group"
   - Select "cluster-admins"
   - Click "Join"

3. Verify group claim in token:
   - Navigate to: Clients → <your-eem-client>
   - Click "Client scopes" tab
   - Ensure "groups" mapper is enabled
   - Test token to verify /groups claim is present
```

### 3. Configure EEM User Groups

After EEM restarts and user logs in again:

```
1. Login to EEM UI
2. Navigate to: Manage → User groups → Cluster maintainers
3. Click "Add user group"
4. Select "cluster-admins" group
5. Click "Next"
6. Select the Event Streams cluster
7. Click "Save"
```

### 4. Verify Functionality

Test that cluster deletion now works:

```
1. Navigate to: Manage → Clusters
2. Click delete icon on Event Streams cluster
3. Confirm deletion
4. Should succeed (no more 403 Forbidden)
```

## Permission Model

### Roles (authorizationClaimPointer)
- **viewer**: View catalog and shared resources (read-only)
- **author**: Create and share resources (includes viewer permissions)
- **admin**: Configure gateways, security, and user groups (includes author permissions)

### User Groups (userGroupClaimPointer)
- **Option Viewers**: View and subscribe to options
- **Event Source Editors**: Create/publish options, edit event sources
- **Cluster Maintainers**: Edit cluster configurations, **DELETE clusters**

### Complete Admin Access

Users with "admin" role in Keycloak token receive:
- **admin** role: Gateway configuration, security controls, user group management
- **author** role: Create/share resources, manage event sources
- **viewer** role: View all UI elements and shared resources

This ensures complete visibility and access to all EEM features.

### Key Differences

| Operation | Requires Role | Requires Group |
|-----------|--------------|----------------|
| Login to EEM | viewer/author/admin | - |
| View all UI menus | admin+author+viewer | - |
| Manage user groups | admin | - |
| Create cluster | author/admin | - |
| Edit cluster | - | Cluster Maintainer |
| **Delete cluster** | - | **Cluster Maintainer** |
| Create event source | author/admin | - |
| Edit event source | - | Event Source Editor |
| Publish options | - | Event Source Editor |

## ROSA Migration Notes

When migrating to ROSA with Azure AD:

```yaml
authConfig:
  authType: OIDC
  oidcConfig:
    # Update to Azure AD issuer
    site: https://login.microsoftonline.com/<tenant-id>/v2.0
    
    # Azure AD typically uses /roles for roles
    authorizationClaimPointer: /roles
    
    # Azure AD typically uses /groups for groups
    userGroupClaimPointer: /groups
    
    # Regenerate secret with Azure AD credentials
    secretName: oidc-client-secret
```

Update user-roles.yaml to match Azure AD role names:
```json
{
  "mappings": [
    {
      "id": "EEM-Admin",  // Match Azure AD role name
      "roles": ["admin", "author", "viewer"]
    }
  ]
}
```

## References

- [EEM User Groups Documentation](https://ibm.github.io/event-automation/eem/security/groups/)
- [EEM Cluster Maintainers Documentation](https://ibm.github.io/event-automation/eem/describe/cluster-maintainers/)
- [EEM Managing Access Documentation](https://ibm.github.io/event-automation/eem/security/managing-access/)
- [EEM User Roles Documentation](https://ibm.github.io/event-automation/eem/security/user-roles/)

## Troubleshooting

**User groups menu still not appearing:**
- Verify EEM pod restarted after config change
- Check user logged out and back in
- Verify Keycloak token contains `/groups` claim
- Check EEM logs for OIDC errors
- Verify user has "admin" role in Keycloak token

**Still getting 403 on cluster deletion:**
- Verify user is member of a Keycloak group
- Verify group is assigned as cluster maintainer in EEM
- Verify cluster is assigned to that group
- User must log out and back in after group changes

**Admin user not seeing all UI features:**
- Verify user-roles.yaml assigns all three roles: ["admin", "author", "viewer"]
- Check that user has "admin" in their Keycloak token roles
- Restart EEM pods if role mapping was recently changed
- Clear browser cache and re-login

**Admin API not seeing groups:**
- Add `additionalScopes: ["offline_access"]` to config
- Verify OIDC provider supports offline_access scope
- Regenerate access tokens after config change
