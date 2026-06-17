# OVGS Test Plan

> Based on `ovgs.proto` (v1) and `docs/service-overview.md`

---

## Section 1: Group Management

### 1.1 CreateGroup

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-GROUP-001 | Create a child group under the root org | CreateGroup basic functionality | 1. Authenticate as admin user of org-acmeco<br>2. Call CreateGroup with `parent=org-acmeco`, `description=default`<br>3. Verify response contains a valid `group_id` prefix `group-` | Response returns `group_id` starting with `group-` |
| TC-GROUP-002 | Create a child group under an existing group | CreateGroup nested hierarchy | 1. Authenticate as admin user<br>2. Call CreateGroup with `parent=group-3e7e...`, `description=siteA`<br>3. Verify response contains a new `group_id` | New group created as child of the specified parent group |
| TC-GROUP-003 | Reject creation with empty parent | CreateGroup validation | 1. Call CreateGroup with `parent=""`, `description=test` | Returns `INVALID_ARGUMENT` error |
| TC-GROUP-004 | Reject creation with empty description | CreateGroup validation | 1. Call CreateGroup with `parent=org-acmeco`, `description=""` | Returns `INVALID_ARGUMENT` error |
| TC-GROUP-005 | Reject creation with non-existent parent | CreateGroup validation | 1. Call CreateGroup with `parent=org-nonexistent`, `description=test` | Returns `NOT_FOUND` error |
| TC-GROUP-006 | Reject duplicate group name under same parent | CreateGroup uniqueness constraint | 1. Call CreateGroup with `parent=org-acmeco`, `description=default`<br>2. Call CreateGroup again with same `parent` and `description` | Second call returns `ALREADY_EXISTS` error |
| TC-GROUP-007 | Reject creation by non-admin user | CreateGroup role authorization | 1. Authenticate as ASSIGNER user<br>2. Call CreateGroup | Returns `PERMISSION_DENIED` error |
| TC-GROUP-008 | Verify delegated_orgs inheritance from parent | CreateGroup delegation inheritance | 1. Add delegated org `org-google` to parent group via AddGroupDelegatedOrg<br>2. Call CreateGroup with that parent<br>3. Call GetGroup on the newly created child group | Child group's `delegated_orgs` contains `org-google` inherited from parent |
| TC-GROUP-009 | Allow duplicate description under different parents | CreateGroup scope uniqueness | 1. Create group with `description=siteA` under parent group A<br>2. Create group with `description=siteA` under parent group B | Both calls succeed; description uniqueness is scoped to same parent only |

### 1.2 GetGroup

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-GROUP-010 | Retrieve root group details | GetGroup for root group | 1. Call GetGroup with `group_id=org-acmeco`<br>2. Verify all fields populated | Response contains `group_id`, `cert_ids`, `components`, `users`, `child_group_ids`, `delegated_orgs` |
| TC-GROUP-011 | Retrieve child group details | GetGroup for child group | 1. Call GetGroup with a valid child `group_id`<br>2. Verify response fields | Response shows correct `cert_ids`, `components`, `users`, `child_group_ids` |
| TC-GROUP-012 | Reject retrieval of non-existent group | GetGroup validation | 1. Call GetGroup with `group_id=group-nonexistent` | Returns `NOT_FOUND` error |
| TC-GROUP-013 | Reject access by unauthorized user | GetGroup authorization | 1. Authenticate as user who does not belong to the target group or its ancestors<br>2. Call GetGroup | Returns `PERMISSION_DENIED` error |
| TC-GROUP-014 | Delegated caller sees only own org users | GetGroup delegation filtering | 1. Authenticate as delegated user from org-google<br>2. Call GetGroup on a group with mixed-org users | Response `users` only contains users from org-google; `delegated_orgs` only shows org-google entry |
| TC-GROUP-015 | Owner org caller sees all users | GetGroup full visibility | 1. Authenticate as owner org admin<br>2. Call GetGroup on group with delegated users | Response `users` includes all users from all orgs; `delegated_orgs` shows full map |

### 1.3 DeleteGroup

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-GROUP-016 | Delete an empty child group | DeleteGroup basic functionality | 1. Create a child group with no users, certs, serials, or child groups<br>2. Call DeleteGroup with that `group_id` | Group is deleted successfully; subsequent GetGroup returns `NOT_FOUND` |
| TC-GROUP-017 | Reject deletion of root group | DeleteGroup root protection | 1. Call DeleteGroup with `group_id=org-acmeco` | Returns `INVALID_ARGUMENT` error |
| TC-GROUP-018 | Reject deletion with empty group_id | DeleteGroup validation | 1. Call DeleteGroup with `group_id=""` | Returns `INVALID_ARGUMENT` error |
| TC-GROUP-019 | Reject deletion of non-existent group | DeleteGroup validation | 1. Call DeleteGroup with `group_id=group-nonexistent` | Returns `NOT_FOUND` error |
| TC-GROUP-020 | Reject deletion when group has child groups | DeleteGroup precondition | 1. Create a parent group and a child group under it<br>2. Call DeleteGroup on the parent group | Returns `FAILED_PRECONDITION` error |
| TC-GROUP-021 | Reject deletion when group has users | DeleteGroup precondition | 1. Add a user to the group<br>2. Call DeleteGroup | Returns `FAILED_PRECONDITION` error |
| TC-GROUP-022 | Reject deletion when group has components | DeleteGroup precondition | 1. Assign a serial to the group<br>2. Call DeleteGroup | Returns `FAILED_PRECONDITION` error |
| TC-GROUP-023 | Reject deletion when group has certificates | DeleteGroup precondition | 1. Create a domain cert in the group<br>2. Call DeleteGroup | Returns `FAILED_PRECONDITION` error |
| TC-GROUP-024 | Reject deletion by non-admin user | DeleteGroup authorization | 1. Authenticate as ASSIGNER user<br>2. Call DeleteGroup | Returns `PERMISSION_DENIED` error |

---

## Section 2: Delegation Management

### 2.1 AddGroupDelegatedOrg

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-DELEG-001 | Add a delegated org to a group | AddGroupDelegatedOrg basic functionality | 1. Authenticate as owner org root group admin<br>2. Call AddGroupDelegatedOrg with `group_id=group-xxx`, `org_id=org-google`<br>3. Call GetGroup to verify | Group's `delegated_orgs` includes `org-google` |
| TC-DELEG-002 | Add wildcard org for open delegation | AddGroupDelegatedOrg wildcard support | 1. Call AddGroupDelegatedOrg with `org_id="*"`<br>2. Call GetGroup to verify | Group's `delegated_orgs` contains key `"*"` |
| TC-DELEG-003 | Reject adding own org as delegated org | AddGroupDelegatedOrg self-delegation prevention | 1. Call AddGroupDelegatedOrg with `org_id` equal to the group's owner org | Returns `INVALID_ARGUMENT` error |
| TC-DELEG-004 | Reject adding already-existing delegated org | AddGroupDelegatedOrg deduplication | 1. Add org-google as delegated org<br>2. Call AddGroupDelegatedOrg again with same `org_id` | Returns `ALREADY_EXISTS` error |
| TC-DELEG-005 | Reject with empty group_id | AddGroupDelegatedOrg validation | 1. Call AddGroupDelegatedOrg with `group_id=""` | Returns `INVALID_ARGUMENT` error |
| TC-DELEG-006 | Reject with empty org_id | AddGroupDelegatedOrg validation | 1. Call AddGroupDelegatedOrg with `org_id=""` | Returns `INVALID_ARGUMENT` error |
| TC-DELEG-007 | Reject by non-root-group admin | AddGroupDelegatedOrg authorization scope | 1. Authenticate as admin of a non-root group<br>2. Call AddGroupDelegatedOrg | Returns `PERMISSION_DENIED` error |
| TC-DELEG-008 | Reject by admin of non-owner org | AddGroupDelegatedOrg cross-org restriction | 1. Authenticate as admin from a different owner org<br>2. Call AddGroupDelegatedOrg | Returns `PERMISSION_DENIED` error |
| TC-DELEG-009 | Reject by ASSIGNER user | AddGroupDelegatedOrg role restriction | 1. Authenticate as ASSIGNER<br>2. Call AddGroupDelegatedOrg | Returns `PERMISSION_DENIED` error |

### 2.2 RemoveGroupDelegatedOrg

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-DELEG-010 | Remove a delegated org from a group | RemoveGroupDelegatedOrg basic functionality | 1. Ensure org-google is in delegated_orgs and no delegated users from org-google remain<br>2. Call RemoveGroupDelegatedOrg<br>3. Call GetGroup to verify | org-google removed from `delegated_orgs` |
| TC-DELEG-011 | Reject removal when delegated org users still exist | RemoveGroupDelegatedOrg precondition | 1. Add delegated users from org-google to the group<br>2. Call RemoveGroupDelegatedOrg for org-google | Returns `FAILED_PRECONDITION` error |
| TC-DELEG-012 | Reject self-delegation removal | RemoveGroupDelegatedOrg self-org check | 1. Call RemoveGroupDelegatedOrg with `org_id` equal to group's owner org | Returns `INVALID_ARGUMENT` error |
| TC-DELEG-013 | Reject with empty fields | RemoveGroupDelegatedOrg validation | 1. Call RemoveGroupDelegatedOrg with empty `group_id` or `org_id` | Returns `INVALID_ARGUMENT` error |
| TC-DELEG-014 | Reject removal of non-existent delegated org | RemoveGroupDelegatedOrg validation | 1. Call RemoveGroupDelegatedOrg with `org_id=org-notdelegated` | Returns `NOT_FOUND` error |
| TC-DELEG-015 | Reject by non-root-group admin | RemoveGroupDelegatedOrg authorization | 1. Authenticate as child-group admin<br>2. Call RemoveGroupDelegatedOrg | Returns `PERMISSION_DENIED` error |

---

## Section 3: User Role Management

### 3.1 AddUserRole

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-USER-001 | Add a USER with ADMIN role to a group | AddUserRole basic functionality | 1. Call AddUserRole with `username=user1`, `user_type=ACCOUNT_TYPE_USER`, `org_id=org-acmeco`, `group_id=group-xxx`, `user_role=USER_ROLE_ADMIN`<br>2. Call GetGroup to verify | User appears in group's user list with ADMIN role |
| TC-USER-002 | Add a SERVICE_ACCOUNT with ADMIN role | AddUserRole service account support | 1. Call AddUserRole with `username=srv-admin`, `user_type=ACCOUNT_TYPE_SERVICE_ACCOUNT`, `org_id=org-acmeco`, `group_id=org-acmeco`, `user_role=USER_ROLE_ADMIN` | Service account appears in root group's user list |
| TC-USER-003 | Add user with REQUESTOR role | AddUserRole role assignment | 1. Call AddUserRole with `user_role=USER_ROLE_REQUESTOR` | User added with REQUESTOR role |
| TC-USER-004 | Add user with ASSIGNER role | AddUserRole role assignment | 1. Call AddUserRole with `user_role=USER_ROLE_ASSIGNER` | User added with ASSIGNER role |
| TC-USER-005 | Reject adding duplicate user to same group | AddUserRole deduplication | 1. Add user to group<br>2. Call AddUserRole again with same parameters | Returns `ALREADY_EXISTS` error |
| TC-USER-006 | Reject with empty username | AddUserRole validation | 1. Call AddUserRole with `username=""` | Returns `INVALID_ARGUMENT` error |
| TC-USER-007 | Reject with non-existent group | AddUserRole validation | 1. Call AddUserRole with `group_id=group-nonexistent` | Returns `NOT_FOUND` error |
| TC-USER-008 | Owner org admin can add delegated user | AddUserRole cross-org with delegation | 1. Ensure org-google is in group's delegated_orgs<br>2. Call AddUserRole with `org_id=org-google` for a user from org-google | Delegated user added successfully |
| TC-USER-009 | Reject adding delegated user when org not in delegated_orgs | AddUserRole delegation enforcement | 1. Ensure org-google is NOT in group's delegated_orgs<br>2. Call AddUserRole with `org_id=org-google` | Returns `PERMISSION_DENIED` or appropriate error |
| TC-USER-010 | Delegated caller can only add users from own org | AddUserRole delegation scope | 1. Authenticate as delegated admin from org-google<br>2. Call AddUserRole with `org_id=org-acmeco` (different org) | Returns `PERMISSION_DENIED` error |
| TC-USER-011 | Delegated caller can add users from own org | AddUserRole delegation scope | 1. Authenticate as delegated admin from org-google<br>2. Call AddUserRole with `org_id=org-google` for a user from org-google | User added successfully |
| TC-USER-012 | ASSIGNER can assign ASSIGNER role | AddUserRole role delegation by ASSIGNER | 1. Authenticate as ASSIGNER<br>2. Call AddUserRole with `user_role=USER_ROLE_ASSIGNER` | User added with ASSIGNER role |
| TC-USER-013 | ASSIGNER can assign REQUESTOR role | AddUserRole role delegation by ASSIGNER | 1. Authenticate as ASSIGNER<br>2. Call AddUserRole with `user_role=USER_ROLE_REQUESTOR` | User added with REQUESTOR role |
| TC-USER-014 | ASSIGNER cannot assign ADMIN role | AddUserRole role delegation boundary | 1. Authenticate as ASSIGNER<br>2. Call AddUserRole with `user_role=USER_ROLE_ADMIN` | Returns `PERMISSION_DENIED` error |
| TC-USER-015 | REQUESTOR cannot assign any role | AddUserRole role delegation boundary | 1. Authenticate as REQUESTOR<br>2. Call AddUserRole with any role | Returns `PERMISSION_DENIED` error |

### 3.2 RemoveUserRole

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-USER-016 | Remove a user from a group | RemoveUserRole basic functionality | 1. Add user to group<br>2. Call RemoveUserRole with same user tuple and group_id<br>3. Call GetGroup to verify | User no longer appears in group's user list |
| TC-USER-017 | Reject removal of non-member user | RemoveUserRole validation | 1. Call RemoveUserRole for a user not in the group | Returns `NOT_FOUND` error |
| TC-USER-018 | Reject with empty fields | RemoveUserRole validation | 1. Call RemoveUserRole with empty `username` or `group_id` | Returns `INVALID_ARGUMENT` error |
| TC-USER-019 | Delegated caller can only remove own-org users | RemoveUserRole delegation scope | 1. Authenticate as delegated admin from org-google<br>2. Call RemoveUserRole for a user from org-acmeco | Returns `PERMISSION_DENIED` error |
| TC-USER-020 | Delegated caller can remove own-org users | RemoveUserRole delegation scope | 1. Authenticate as delegated admin from org-google<br>2. Call RemoveUserRole for a user from org-google | User removed successfully |
| TC-USER-021 | Reject removal by ASSIGNER | RemoveUserRole authorization | 1. Authenticate as ASSIGNER<br>2. Call RemoveUserRole | Returns `PERMISSION_DENIED` error |

### 3.3 GetUserRole

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-USER-022 | View roles of a user across groups | GetUserRole basic functionality | 1. Add user to multiple groups with different roles<br>2. Call GetUserRole with user tuple | Response `groups` map contains all groups and corresponding roles |
| TC-USER-023 | View roles returns empty for non-member groups | GetUserRole scope filtering | 1. Authenticate as user A who has no group overlap with user B<br>2. Call GetUserRole for user B | Response `groups` map is empty |
| TC-USER-024 | Delegated caller cannot view owner org user roles | GetUserRole delegation restriction | 1. Authenticate as delegated user from org-google<br>2. Call GetUserRole for a user from org-acmeco | Response `groups` map is empty |
| TC-USER-025 | Delegated caller can view own-org user roles | GetUserRole delegation scope | 1. Authenticate as delegated user from org-google<br>2. Call GetUserRole for a user from org-google | Response shows roles in groups visible to the caller |
| TC-USER-026 | Reject with empty username | GetUserRole validation | 1. Call GetUserRole with `username=""` | Returns `INVALID_ARGUMENT` error |
| TC-USER-027 | Reject for non-existent user | GetUserRole validation | 1. Call GetUserRole for a user tuple that does not exist | Returns `NOT_FOUND` error |

---

## Section 4: Domain Certificate Management

### 4.1 CreateDomainCert

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-CERT-001 | Create a domain cert in a group | CreateDomainCert basic functionality | 1. Call CreateDomainCert with `group_id`, valid `certificate_der`, `revocation_checks=true`, `expiry_time` in the future<br>2. Verify response contains `cert_id` | Returns `cert_id` starting with `cert-` |
| TC-CERT-002 | Reject with past expiry_time | CreateDomainCert time validation | 1. Call CreateDomainCert with `expiry_time` in the past | Returns `INVALID_ARGUMENT` error |
| TC-CERT-003 | Reject with empty expiry_time | CreateDomainCert validation | 1. Call CreateDomainCert with `expiry_time` not set | Returns `INVALID_ARGUMENT` error |
| TC-CERT-004 | Reject with invalid certificate | CreateDomainCert cert validation | 1. Call CreateDomainCert with malformed `certificate_der` | Returns `INVALID_ARGUMENT` error |
| TC-CERT-005 | Reject duplicate cert tuple in same group | CreateDomainCert deduplication | 1. CreateDomainCert with (cert_der, revocation, expiry) tuple<br>2. Call CreateDomainCert again with identical tuple in same group | Returns `ALREADY_EXISTS` error |
| TC-CERT-006 | Allow same cert tuple in different groups | CreateDomainCert group scoping | 1. CreateDomainCert in group A<br>2. CreateDomainCert with same tuple in group B | Second call succeeds; each group gets its own `cert_id` |
| TC-CERT-007 | Reject with non-existent group | CreateDomainCert validation | 1. Call CreateDomainCert with `group_id=group-nonexistent` | Returns `NOT_FOUND` error |
| TC-CERT-008 | Reject by REQUESTOR | CreateDomainCert authorization | 1. Authenticate as REQUESTOR<br>2. Call CreateDomainCert | Returns `PERMISSION_DENIED` error |
| TC-CERT-009 | Allow creation by ASSIGNER | CreateDomainCert authorization | 1. Authenticate as ASSIGNER<br>2. Call CreateDomainCert | Certificate created successfully |

### 4.2 GetDomainCert

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-CERT-010 | Retrieve domain cert details | GetDomainCert basic functionality | 1. Create a domain cert<br>2. Call GetDomainCert with returned `cert_id` | Response contains `cert_id`, `group_id`, `certificate_der`, `revocation_checks`, `expiry_time` matching the creation request |
| TC-CERT-011 | Reject with non-existent cert_id | GetDomainCert validation | 1. Call GetDomainCert with `cert_id=cert-nonexistent` | Returns `NOT_FOUND` error |
| TC-CERT-012 | Reject access by user without group permission | GetDomainCert authorization | 1. Authenticate as user with no access to the cert's group<br>2. Call GetDomainCert | Returns `PERMISSION_DENIED` error |

### 4.3 DeleteDomainCert

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-CERT-013 | Delete a domain cert | DeleteDomainCert basic functionality | 1. Create a domain cert<br>2. Call DeleteDomainCert with `cert_id`<br>3. Call GetDomainCert to verify deletion | Certificate deleted; subsequent GetDomainCert returns `NOT_FOUND` |
| TC-CERT-014 | Reject deletion of non-existent cert | DeleteDomainCert validation | 1. Call DeleteDomainCert with `cert_id=cert-nonexistent` | Returns `NOT_FOUND` error |
| TC-CERT-015 | Reject deletion by REQUESTOR | DeleteDomainCert authorization | 1. Authenticate as REQUESTOR<br>2. Call DeleteDomainCert | Returns `PERMISSION_DENIED` error |
| TC-CERT-016 | Allow deletion by ASSIGNER | DeleteDomainCert authorization | 1. Authenticate as ASSIGNER<br>2. Call DeleteDomainCert | Certificate deleted successfully |

---

## Section 5: Serial Number Management

### 5.1 AddSerial

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-SERIAL-001 | Assign a serial to a child group | AddSerial basic functionality | 1. Ensure serial ABC101 exists in root group<br>2. Call AddSerial with `component={ien=30065, serial_number=ABC101}`, `group_id=group-xxx`<br>3. Call GetSerial to verify | Serial appears in both root group and the assigned child group |
| TC-SERIAL-002 | Reassign serial from one group to another | AddSerial reassignment behavior | 1. Assign serial to group A<br>2. Call AddSerial with same serial and group B<br>3. Call GetSerial | Serial removed from group A, now appears in root group and group B only |
| TC-SERIAL-003 | Reject assigning serial already in the same group | AddSerial idempotency check | 1. Assign serial to group A<br>2. Call AddSerial with same serial and same group A | Returns `ALREADY_EXISTS` error |
| TC-SERIAL-004 | Reject with empty component fields | AddSerial validation | 1. Call AddSerial with `component={ien="", serial_number=""}` | Returns `INVALID_ARGUMENT` error |
| TC-SERIAL-005 | Reject with non-existent group | AddSerial validation | 1. Call AddSerial with `group_id=group-nonexistent` | Returns `NOT_FOUND` error |
| TC-SERIAL-006 | Reject with non-existent serial | AddSerial validation | 1. Call AddSerial with a serial number not in the root group | Returns `NOT_FOUND` error |
| TC-SERIAL-007 | Reject when caller lacks source group permission | AddSerial dual-permission check | 1. User has permission to group B but not root group<br>2. Serial only exists in root group<br>3. Call AddSerial | Returns `PERMISSION_DENIED` error |
| TC-SERIAL-008 | Reject when caller lacks destination group permission | AddSerial dual-permission check | 1. User has permission to root group but not group B<br>2. Call AddSerial to move serial to group B | Returns `PERMISSION_DENIED` error |
| TC-SERIAL-009 | Reject with invalid IEN | AddSerial IEN validation | 1. Call AddSerial with `ien=99999` (not applicable for voucher issuer) | Returns `INVALID_ARGUMENT` error |
| TC-SERIAL-010 | Reject by REQUESTOR | AddSerial authorization | 1. Authenticate as REQUESTOR<br>2. Call AddSerial | Returns `PERMISSION_DENIED` error |

### 5.2 RemoveSerial

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-SERIAL-011 | Remove a serial from a child group | RemoveSerial basic functionality | 1. Assign serial to group A<br>2. Call RemoveSerial with same component and group A<br>3. Call GetSerial to verify | Serial no longer appears in group A; still in root group |
| TC-SERIAL-012 | Reject removing serial not in the group | RemoveSerial validation | 1. Call RemoveSerial for a serial not assigned to the specified group | Returns `NOT_FOUND` error |
| TC-SERIAL-013 | Reject with empty fields | RemoveSerial validation | 1. Call RemoveSerial with empty component or group_id | Returns `INVALID_ARGUMENT` error |
| TC-SERIAL-014 | Reject by REQUESTOR | RemoveSerial authorization | 1. Authenticate as REQUESTOR<br>2. Call RemoveSerial | Returns `PERMISSION_DENIED` error |

### 5.3 GetSerial

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-SERIAL-015 | Retrieve serial details with TPM info | GetSerial basic functionality | 1. Call GetSerial with `component={ien=30065, serial_number=ABC101}` | Response contains `group_ids`, `mac_addr`, `model`, `tpm_info` (if applicable) |
| TC-SERIAL-016 | Verify TPM 1.2 endorsement_key field | GetSerial TPM version differentiation | 1. Query a TPM 1.2 device serial<br>2. Check `tpm_info.ek_data` variant | `endorsement_key` field is populated (bytes) |
| TC-SERIAL-017 | Verify TPM 2.0 ek_certificate field | GetSerial TPM version differentiation | 1. Query a TPM 2.0 + IDevID device serial<br>2. Check `tpm_info.ek_data` variant | `ek_certificate` field is populated (DER encoded X.509) |
| TC-SERIAL-018 | Verify TPM 2.0 platform_primary_key field | GetSerial TPM version differentiation | 1. Query a TPM 2.0 device without IDevID<br>2. Check `tpm_info.platform_primary_key` | `platform_primary_key` field is populated |
| TC-SERIAL-019 | Reject with non-existent component | GetSerial validation | 1. Call GetSerial with `component={ien=30065, serial_number=NONEXISTENT}` | Returns `NOT_FOUND` error |
| TC-SERIAL-020 | Reject with invalid IEN | GetSerial validation | 1. Call GetSerial with `ien=99999` | Returns `INVALID_ARGUMENT` error |
| TC-SERIAL-021 | Reject by unauthorized user | GetSerial authorization | 1. Authenticate as user with no access to serial's group<br>2. Call GetSerial | Returns `PERMISSION_DENIED` error |

### 5.4 GetSerials (Batch)

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-SERIAL-022 | Batch retrieve multiple serials | GetSerials basic functionality | 1. Call GetSerials with multiple components<br>2. Verify response | Response `serials` list matches the order of request components |
| TC-SERIAL-023 | Verify response order preservation | GetSerials ordering guarantee | 1. Call GetSerials with components [B, A, C]<br>2. Check response order | Response serials are in order [B, A, C] matching request |

### 5.5 GetComponents (Server Streaming)

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-SERIAL-024 | Stream all components in a group | GetComponents basic functionality | 1. Call GetComponents with `group_id` of a group containing multiple components<br>2. Collect all streamed responses | Each streamed message contains a Component and its GetSerialResponse data |
| TC-SERIAL-025 | Reject with non-existent group | GetComponents validation | 1. Call GetComponents with `group_id=group-nonexistent` | Returns `NOT_FOUND` error |
| TC-SERIAL-026 | Reject by unauthorized user | GetComponents authorization | 1. Authenticate as user without access to the group<br>2. Call GetComponents | Returns `PERMISSION_DENIED` error |

---

## Section 6: Ownership Voucher Issuance

### 6.1 GetOwnershipVoucher

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-OV-001 | Issue an ownership voucher for a valid serial | GetOwnershipVoucher basic functionality | 1. Ensure serial ABC101 is in a group with a valid domain cert<br>2. Call GetOwnershipVoucher with `component`, `cert_id`, `lifetime` in future | Response contains `voucher_cms` (non-empty bytes), `tpm_info` (if applicable) |
| TC-OV-002 | Verify voucher CMS is valid RFC8366 format | GetOwnershipVoucher output validation | 1. Request an ownership voucher<br>2. Decode the `voucher_cms` binary CMS<br>3. Verify it contains `serial-number`, `pinned-domain-cert`, `created-on`, `expires-on`, `assertion`, `domain-cert-revocation-checks` | Voucher conforms to RFC8366 §5.3 format |
| TC-OV-003 | Verify different vouchers for same parameters | GetOwnershipVoucher non-idempotent issuance | 1. Request voucher for same (component, cert_id, lifetime)<br>2. Request again with identical parameters<br>3. Compare two `voucher_cms` values | Two vouchers differ but both are valid |
| TC-OV-004 | Reject with lifetime in the past | GetOwnershipVoucher time validation | 1. Call GetOwnershipVoucher with `lifetime` in the past | Returns `INVALID_ARGUMENT` error |
| TC-OV-005 | Reject with lifetime exceeding cert expiry | GetOwnershipVoucher time validation | 1. Create domain cert with expiry_time = 2025-01-01<br>2. Call GetOwnershipVoucher with `lifetime` = 2026-01-01 | Returns `INVALID_ARGUMENT` error |
| TC-OV-006 | Reject when cert and serial belong to different orgs | GetOwnershipVoucher org consistency check | 1. Serial in org-acmeco, cert in org-google<br>2. Call GetOwnershipVoucher | Returns `INVALID_ARGUMENT` error |
| TC-OV-007 | Reject with non-existent component | GetOwnershipVoucher validation | 1. Call GetOwnershipVoucher with a non-existent serial | Returns `FAILED_PRECONDITION` or `INVALID_ARGUMENT` error |
| TC-OV-008 | Reject with non-existent cert_id | GetOwnershipVoucher validation | 1. Call GetOwnershipVoucher with `cert_id=cert-nonexistent` | Returns `FAILED_PRECONDITION` error |
| TC-OV-009 | Reject by user without access to serial's group | GetOwnershipVoucher authorization | 1. Authenticate as user with no access to the serial's group<br>2. Call GetOwnershipVoucher | Returns `PERMISSION_DENIED` error |
| TC-OV-010 | Reject with invalid IEN | GetOwnershipVoucher IEN validation | 1. Call GetOwnershipVoucher with `ien=99999` | Returns `INVALID_ARGUMENT` error |
| TC-OV-011 | Allow REQUESTOR to issue voucher | GetOwnershipVoucher REQUESTOR access | 1. Authenticate as REQUESTOR with access to serial's group<br>2. Call GetOwnershipVoucher | Voucher issued successfully |
| TC-OV-012 | Verify TPM info returned with voucher | GetOwnershipVoucher TPM data | 1. Request voucher for a device with TPM<br>2. Check `tpm_info` in response | `tpm_info` contains `ek_certificate` or `endorsement_key` and optionally `platform_primary_key` |

### 6.2 GetOwnershipVouchers (Batch)

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-OV-013 | Batch issue ownership vouchers | GetOwnershipVouchers basic functionality | 1. Call GetOwnershipVouchers with multiple components, same `cert_id` and `lifetime` | Response contains `ownership_vouchers` list with one entry per component |
| TC-OV-014 | Batch with some invalid components | GetOwnershipVouchers partial failure | 1. Call GetOwnershipVouchers with mix of valid and non-existent components | Entire request fails or returns errors for invalid components (verify implementation behavior) |

---

## Section 7: Hierarchy and Permission Inheritance

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-HIER-001 | User can access child group resources | Subtree access inheritance | 1. Add user to parent group as ADMIN<br>2. Create child group<br>3. User calls GetGroup on child group | User can access child group successfully |
| TC-HIER-002 | User cannot access parent group from child group | Upward isolation | 1. Add user to child group only<br>2. User calls GetGroup on parent group | Returns `PERMISSION_DENIED` error |
| TC-HIER-003 | User can request OV for serial in child group | Hierarchical serial access | 1. Assign serial to child group<br>2. User in parent group with REQUESTOR role calls GetOwnershipVoucher | Voucher issued successfully |
| TC-HIER-004 | User cannot request OV for serial in parent group | Upward serial isolation | 1. Assign serial to parent group<br>2. User in child group with REQUESTOR role calls GetOwnershipVoucher | Returns `PERMISSION_DENIED` error |
| TC-HIER-005 | Overlapping subtree grants highest permission | Permission escalation on overlap | 1. Add user to group A as REQUESTOR<br>2. Add same user to group B (child of A) as ADMIN<br>3. User accesses resources in group B | User has ADMIN access to group B resources |
| TC-HIER-006 | User can belong to multiple non-overlapping groups | Multi-group membership | 1. Add user to group A and group C (separate branches)<br>2. User calls GetGroup on both | User can access both groups with their respective roles |
| TC-HIER-007 | Admin can manage child group structure | Hierarchical group management | 1. Add user as ADMIN to root group<br>2. User creates child group under a child group | Group created successfully |

---

## Section 8: Cross-Org Delegation End-to-End Scenarios

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-E2E-001 | Full delegation workflow | End-to-end delegation setup and usage | 1. Owner org admin adds org-google to delegated_orgs<br>2. Owner org admin adds a delegated user from org-google<br>3. Delegated user calls GetGroup | Delegated user sees only own-org users and restricted delegation info |
| TC-E2E-002 | Delegated user adds another delegated user from same org | Delegation chain within same org | 1. Delegated admin from org-google calls AddUserRole for another org-google user<br>2. New user calls GetGroup | New user added and can access the group |
| TC-E2E-003 | Delegated user cannot add user from different org | Delegation org boundary enforcement | 1. Delegated admin from org-google calls AddUserRole for org-acmeco user | Returns `PERMISSION_DENIED` error |
| TC-E2E-004 | Delegated user requests ownership voucher | Delegation voucher access | 1. Delegated user with REQUESTOR role<br>2. Call GetOwnershipVoucher for a serial in the delegated group | Voucher issued if user has access to the serial's group |
| TC-E2E-005 | Wildcard delegation allows any org | Open delegation with `*` | 1. Add `org_id="*"` to group's delegated_orgs<br>2. Add user from org-other (not explicitly listed)<br>3. User accesses the group | User from any org can be added and access the group |
| TC-E2E-006 | Remove delegation org after cleaning up users | Delegation teardown | 1. Remove all delegated users from org-google<br>2. Call RemoveGroupDelegatedOrg for org-google<br>3. Verify via GetGroup | org-google removed from delegated_orgs; delegated users can no longer access |

---

## Section 9: Error Handling and Edge Cases

| Case ID | Test Purpose | Test Function Point | Detailed Steps | Expected Result |
|---------|-------------|---------------------|----------------|-----------------|
| TC-ERR-001 | Unauthenticated request is rejected | Authentication enforcement | 1. Call any RPC without a valid access token | Returns `UNAUTHENTICATED` error |
| TC-ERR-002 | Expired token is rejected | Token expiry handling | 1. Call any RPC with an expired access token | Returns `UNAUTHENTICATED` error |
| TC-ERR-003 | Deprecated public_key_der still populated | Backward compatibility | 1. Call GetSerial for a device with TPM<br>2. Check `public_key_der` field | `public_key_der` is populated (deprecated but functional) |
| TC-ERR-004 | Deprecated public_key_der in voucher response | Backward compatibility | 1. Call GetOwnershipVoucher<br>2. Check `public_key_der` field | `public_key_der` is populated (deprecated but functional) |
| TC-ERR-005 | SUPPORT role can invoke all RPCs | SUPPORT full access | 1. Authenticate as SUPPORT user<br>2. Call CreateGroup, AddUserRole, AddGroupDelegatedOrg, etc. | All RPCs succeed |
| TC-ERR-006 | Concurrent voucher requests for same serial | Concurrency handling | 1. Send two simultaneous GetOwnershipVoucher requests for the same serial<br>2. Verify both responses | Both requests succeed; each returns a distinct valid voucher |
| TC-ERR-007 | Delete group with all resources removed step by step | Full group cleanup | 1. Create group with users, certs, serials, and child groups<br>2. Remove child groups<br>3. Remove serials<br>4. Delete certs<br>5. Remove users<br>6. Call DeleteGroup | Group deleted successfully after all resources removed |
| TC-ERR-008 | Serial lifecycle: root → child group → another child group → remove | Serial reassignment chain | 1. Serial starts in root group<br>2. AddSerial to group A<br>3. AddSerial to group B<br>4. RemoveSerial from group B<br>5. Verify GetSerial at each step | Serial always in root group; at most in one additional group; reassignment works correctly |
