# Odoo 18.0 → 19.0 Migration Guide

**Created:** 2026-02-03 by Jose (learned the hard way)

This guide documents breaking changes discovered while porting OCA modules from Odoo 18.0 to 19.0 CE.

---

## Breaking Changes

### 1. SQL Constraints — `_sql_constraints` DEPRECATED

**Old (18.0):**
```python
_sql_constraints = [
    ("name_unique", "UNIQUE(name)", "Name must be unique!"),
]
```

**New (19.0):**
```python
class MyModel(models.Model):
    _name = "my.model"
    
    # Class attribute with underscore prefix
    _name_unique = models.Constraint(
        "UNIQUE(name)",
        "Name must be unique!",
    )
```

**Key differences:**
- Use `models.Constraint()` as a class attribute
- Attribute name should be descriptive with underscore prefix
- First arg is SQL constraint, second is error message
- No name parameter — the attribute name becomes the constraint name

---

### 2. res.groups — `category_id` Field REMOVED

**Old (18.0):**
```xml
<record id="my_group" model="res.groups">
    <field name="name">My Group</field>
    <field name="category_id" ref="base.module_category_hidden"/>
</record>
```

**New (19.0):**
```xml
<record id="my_group" model="res.groups">
    <field name="name">My Group</field>
    <!-- category_id no longer exists — just remove it -->
</record>
```

**Note:** `ir.module.category` still exists, but `res.groups` no longer links to it.

---

### 3. res.groups — `users` Field REMOVED

**Old (18.0):**
```xml
<record id="my_group" model="res.groups">
    <field name="name">My Group</field>
    <field name="users" eval="[(4, ref('base.user_admin'))]"/>
</record>
```

**New (19.0):**
```xml
<record id="my_group" model="res.groups">
    <field name="name">My Group</field>
    <!-- users field removed — assign users via UI or res_groups_users_rel table -->
</record>
```

**Workaround:** Assign users to groups via:
- Odoo UI after install
- Direct SQL: `INSERT INTO res_groups_users_rel (uid, gid) VALUES (...)`

---

### 4. ir.actions.act_window — `target='inline'` INVALID

**Old (18.0):**
```xml
<field name="target">inline</field>
```

**New (19.0):**
```xml
<field name="target">new</field>
<!-- or: current, main, fullscreen -->
```

**Valid target values in 19.0:** `current`, `new`, `main`, `fullscreen`

---

### 5. res.users — `groups_id` Field REMOVED from create()

**Old (18.0):**
```python
user = env["res.users"].create({
    "name": "Test User",
    "login": "test@example.com",
    "groups_id": [(4, admin_group_id)],
})
```

**New (19.0):**
```python
# Create user first
user = env["res.users"].create({
    "name": "Test User", 
    "login": "test@example.com",
})
# Then assign groups via SQL
env.cr.execute(
    "INSERT INTO res_groups_users_rel (uid, gid) VALUES (%s, %s)",
    (user.id, admin_group_id)
)
```

---

### 6. Command Syntax in XML

**Old (18.0):**
```xml
<field name="users" eval="[(4, ref('base.user_admin'))]"/>
```

**New (19.0):**
```xml
<field name="implied_ids" eval="[(4, ref('other_group'))]"/>
<!-- Note: some fields still accept this syntax, but 'users' doesn't -->
```

The `(4, id)` tuple syntax (ORM commands) still works for some fields like `implied_ids`, but not for `users`.

---

## Migration Checklist

When porting an OCA module from 18.0 to 19.0:

- [ ] **Bump version** in `__manifest__.py` to `19.0.x.x.x`
- [ ] **Remove migration scripts** in `migrations/` folder (fresh install)
- [ ] **Search for `_sql_constraints`** → convert to `models.Constraint()`
- [ ] **Search security XMLs for `category_id`** → remove
- [ ] **Search security XMLs for `users` field** → remove
- [ ] **Search for `target.*inline`** → change to `new` or `current`
- [ ] **Test module load** with `odoo -i module_name --stop-after-init`
- [ ] **Check logs for warnings** about deprecated features
- [ ] **Test basic functionality** in Odoo UI

---

## Testing Commands

```bash
# Test module installation (from inside Odoo container)
odoo --db_host=db --db_user=odoo --db_password=$PASSWORD \
    -d ovyinc -i module_name --stop-after-init --no-http

# Update existing module
odoo --db_host=db --db_user=odoo --db_password=$PASSWORD \
    -d ovyinc -u module_name --stop-after-init --no-http
```

---

## References

- Odoo 19 Release Notes: https://www.odoo.com/documentation/19.0/
- OCA Migration Guide: https://github.com/OCA/maintainer-tools/wiki/Migration-to-version-19.0
- Archon KB: Odoo 19 docs ingested (4,912 pages)

---

## Modules Successfully Ported

| Module | Source | Commits | Status |
|--------|--------|---------|--------|
| agreement | OCA/agreement | 4 fixes | ✅ Working |
| agreement_legal | OCA/agreement | 4 fixes | ✅ Working |
| agreement_serviceprofile | OCA/agreement | inherited | ✅ Working |
| agreement_sale | OCA/agreement | inherited | ✅ Working |

## Modules Pending

| Module | Blocker |
|--------|---------|
| sign_oca | Depends on `base_sparse_field` (needs porting) |

---

*Last updated: 2026-02-03*
