# Helpdesk - Forked App for BLKSHP Customization

**App:** Helpdesk (forked for customization)
**Repository:** git@github.com:ericirby/helpdesk.git
**Upstream:** https://github.com/frappe/helpdesk
**Purpose:** Customer support and ticketing system for BLKSHP operations
**Bench:** `/Users/Eric/Development/BLKSHP/BLKSHP-DEV`

---

## Overview

This is a forked instance of Frappe Helpdesk, customized for BLKSHP-specific support workflows. While based on upstream Frappe Helpdesk, this fork may contain customizations specific to hospitality management support needs.

### Fork Information

- **Forked From:** https://github.com/frappe/helpdesk
- **Fork Repository:** https://github.com/ericirby/helpdesk
- **Documentation:** https://github.com/frappe/helpdesk/wiki
- **Purpose:** Modern, open-source helpdesk and customer service software

---

## BLKSHP Customizations

### Current Customizations

Document any BLKSHP-specific customizations here:

**Format:**
```
Date: YYYY-MM-DD
Feature: Description of customization
File: path/to/file
Reason: Why this customization was needed
Issue: BLK-XXX or helpdesk issue (if applicable)
```

### Integration with BLKSHP OS

**Planned Integrations:**
- Link support tickets to Companies
- Track property-level support requests
- Integration with task management
- Custom ticket categories for hospitality
- Department-based ticket routing

**Current Status:**
- [ ] Company linking
- [ ] Property-level categorization
- [ ] Task integration
- [ ] Custom workflows

---

## Development Strategy

### 1. Fork Maintenance

**Syncing with Upstream:**
```bash
cd /Users/Eric/Development/BLKSHP/BLKSHP-DEV/apps/helpdesk

# Add upstream remote (if not already added)
git remote add upstream https://github.com/frappe/helpdesk.git

# Fetch upstream changes
git fetch upstream

# Check what's new
git log HEAD..upstream/main --oneline

# Merge upstream (carefully)
git merge upstream/main

# Resolve conflicts if any
# Test thoroughly after merge
```

**After Upstream Merge:**
- Run migrations: `./env/bin/bench --site blkshp.local migrate`
- Test all customizations
- Test BLKSHP integrations
- Run tests: `./env/bin/bench run-tests --app helpdesk`

### 2. Customization Guidelines

**Prefer:**
- Custom hooks in `blkshp_os/hooks.py`
- Custom fields via UI
- Client scripts for form behavior
- Separate DocTypes in `blkshp_os` that link to Helpdesk

**Avoid:**
- Modifying core Helpdesk files (harder to merge upstream)
- Breaking Helpdesk's standard workflow
- Removing upstream features

**Document:**
- All customizations in this file
- Reasons for each customization
- Related Linear issues

### 3. Git Workflow

**Branch Strategy:**
```bash
# Feature branches
git checkout -b feature/blk-XXX-description

# Customization branches (BLKSHP-specific)
git checkout -b custom/blk-XXX-description

# Upstream sync branches
git checkout -b sync/upstream-YYYY-MM-DD
```

**Commit Format:**
```bash
# BLKSHP customizations
git commit -m "feat(custom): add company linking to tickets (BLK-XXX)"

# Upstream merges
git commit -m "chore: merge upstream changes from v1.x.x"

# Bug fixes
git commit -m "fix: resolve ticket assignment issue (BLK-XXX)"
```

---

## Common Helpdesk Patterns

### Working with Tickets

```python
import frappe

# Get ticket
ticket = frappe.get_doc("HD Ticket", "ticket-id")

# Create ticket
ticket = frappe.get_doc({
    "doctype": "HD Ticket",
    "subject": "Issue description",
    "description": "Detailed description",
    "status": "Open",
    "priority": "Medium"
})
ticket.insert()

# Update ticket status
frappe.db.set_value("HD Ticket", "ticket-id", "status", "Resolved")
```

### Custom Fields for BLKSHP

**Add to Tickets:**
- `company` (Link to Company)
- `property` (Link to Property/Entity)
- `department` (Link to Department)
- `ticket_category` (Select: Operations, Accounting, Technical, General)

**Add to Customers:**
- `company_group` (Link to Company Group)
- `is_internal` (Check - for internal BLKSHP staff)

---

## Development Commands

### Helpdesk-Specific Commands

```bash
# Run helpdesk tests
./env/bin/bench run-tests --app helpdesk

# Clear cache
./env/bin/bench --site blkshp.local clear-cache

# Rebuild search index
./env/bin/bench --site blkshp.local rebuild-global-search

# Console (with Helpdesk context)
./env/bin/bench --site blkshp.local console
```

### Git Commands (from bench root)

```bash
# Check fork status
git -C ./apps/helpdesk status

# Check remote configuration
git -C ./apps/helpdesk remote -v

# Fetch upstream
git -C ./apps/helpdesk fetch upstream

# View commit log
git -C ./apps/helpdesk log --oneline -20
```

---

## Troubleshooting

### Fork Sync Issues

**Problem:** Merge conflicts with upstream
**Solution:**
1. Document all customizations first
2. Create backup branch: `git branch backup-before-sync`
3. Merge upstream carefully
4. Test all BLKSHP customizations after merge
5. If issues, revert: `git reset --hard backup-before-sync`

### Migration Failures

```bash
# Check migration logs
tail -f /Users/Eric/Development/BLKSHP/BLKSHP-DEV/logs/bench-start.log

# Force migrate (use carefully)
./env/bin/bench --site blkshp.local migrate --force

# If that fails, reinstall
./env/bin/bench --site blkshp.local reinstall
```

### Installation Issues

```bash
# Reinstall helpdesk
./env/bin/bench --site blkshp.local uninstall-app helpdesk
./env/bin/bench --site blkshp.local install-app helpdesk

# If dependencies are missing
cd apps/helpdesk
npm install
cd ../..
./env/bin/bench build
```

---

## Customization Checklist

When adding BLKSHP-specific features:

- [ ] Document customization in this file
- [ ] Add tests for custom functionality
- [ ] Consider upstream contribution (if generally useful)
- [ ] Test with upstream sync
- [ ] Link to Linear issue
- [ ] Update BLKSHP OS integration if needed
- [ ] Document in user guide (if user-facing)

---

## Upstream Contribution

If you create a feature that could benefit upstream Helpdesk:

1. **Test thoroughly** in BLKSHP environment
2. **Create separate branch** for upstream: `git checkout -b upstream/feature-name`
3. **Remove BLKSHP-specific code** (make it generic)
4. **Follow Frappe contribution guidelines**
5. **Submit PR** to https://github.com/frappe/helpdesk
6. **Link PR** in this document for tracking

---

## Important Notes

### For AI Assistants (Claude)

1. **This is a fork** - treat it differently than upstream apps
2. **Document customizations** before making changes
3. **Test upstream sync** regularly
4. **Prefer hooks** over direct modifications
5. **Link to BLKSHP OS** where possible instead of modifying Helpdesk
6. **Reference Linear issues** in all customizations

### For Developers

1. **Keep fork clean** - minimize divergence from upstream
2. **Sync regularly** - don't let fork get too far behind
3. **Contribute upstream** when features are generally useful
4. **Test thoroughly** - both Helpdesk and BLKSHP features
5. **Document everything** - customizations, reasons, trade-offs

---

## Planned Features

### Phase 1: Basic Integration
- [ ] Link tickets to Companies
- [ ] Link tickets to Departments
- [ ] Custom ticket categories for hospitality
- [ ] Property-level support routing

### Phase 2: Workflow Enhancements
- [ ] Integration with BLKSHP task management
- [ ] Automated ticket creation from review periods
- [ ] SLA tracking for company support
- [ ] Multi-property ticket management

### Phase 3: Analytics
- [ ] Support metrics by company
- [ ] Department-level support analytics
- [ ] Property support trends
- [ ] Custom reporting for management

---

## Resources

- **Upstream Helpdesk:** https://github.com/frappe/helpdesk
- **Helpdesk Documentation:** https://github.com/frappe/helpdesk/wiki
- **Frappe Discuss (Helpdesk):** https://discuss.frappe.io/c/helpdesk
- **Fork Repository:** https://github.com/ericirby/helpdesk

---

## Related Files

- **Bench-level context:** `/Users/Eric/Development/BLKSHP/BLKSHP-DEV/CLAUDE.md`
- **Cursor rules:** `/Users/Eric/Development/BLKSHP/BLKSHP-DEV/.cursor/rules`
- **BLKSHP OS context:** `/Users/Eric/Development/BLKSHP/BLKSHP-DEV/apps/blkshp_os/CLAUDE.md`
- **Frappe context:** `/Users/Eric/Development/BLKSHP/BLKSHP-DEV/apps/frappe/CLAUDE.md`

---

**End of Helpdesk Context Document**

*This document tracks BLKSHP-specific customizations to the forked Helpdesk app. For core Helpdesk functionality, refer to upstream documentation.*
