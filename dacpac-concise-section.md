# Part 8: Database Deployments with .dacpac

## Overview

Our applications use SQL Server Database Projects that generate `.dacpac` (Data-tier Application Package) files for database deployments. A `.dacpac` contains the complete database schema definition and is **state-based** (not migration-based like Flyway/Liquibase).

**Critical Distinction:**
- **State-based:** Defines desired end state, SqlPackage generates diff
- **Not migration-based:** No "up/down" migrations, harder to reverse changes
- **Implication:** Difficult to rollback - must design for forward-fixes

---

## .dacpac and the Build Once, Deploy Everywhere (BODE) Principle

### Can We Achieve BODE with .dacpac?

**YES - but it's "Conditional BODE"**

### What This Means:

**Built Once:**
```
MyDatabase.dacpac (v6.1.0)
├─ Version: 6.1.0
├─ Schema definition (desired state)
└─ Immutable artifact
```

**Promoted Everywhere:**
```
Build → Artifact Registry → DEV → QA → STAGE → PROD
         (Same .dacpac file everywhere - NOT rebuilt)
```

**But Execution Differs:**
```
DEV Database (at v6.0.0):
  SqlPackage compares current vs desired
  Generates: ALTER TABLE Orders ADD FirstName...
  Executes: ~200 lines of SQL

STAGE Database (already at v6.1.0):
  SqlPackage compares current vs desired
  Generates: [No changes needed]
  Executes: 0 lines of SQL

PROD Database (at v6.0.0):
  SqlPackage compares current vs desired
  Generates: Same SQL as DEV
  Executes: ~200 lines of SQL
```

### Comparison Table

| Aspect | Application (Docker) | Database (.dacpac) |
|--------|---------------------|-------------------|
| **Artifact** | Docker image | .dacpac file |
| **Promotion** | Same binary deployed | Same .dacpac deployed |
| **Execution** | Identical per environment | Different SQL per environment |
| **BODE Status** | Pure BODE ✅ | Conditional BODE ⚠️ |
| **Rollback** | Deploy previous tag (2-5 min) | Restore backup (10-30 min + data loss) |

**Is This Acceptable?** YES - Same artifact, consistent end-state, no recompilation.

---

## Rollback Challenges with .dacpac

### Scenario 1: ADDITIVE Schema Changes (Safe Rollback) ✅

**v6.1.0 Changes:**
```sql
ALTER TABLE Orders ADD FirstName NVARCHAR(100) NULL;
ALTER TABLE Orders ADD LastName NVARCHAR(100) NULL;
-- Old CustomerName column still exists
```

**Rollback:**
```bash
# Deploy previous app version
kubectl set image deployment/myapp myapp=myapp:v6.0.0

# Database stays at v6.1.0 (has new columns)
# App v6.0.0 ignores new columns, uses old CustomerName
# ✅ NO DATABASE ROLLBACK NEEDED
# ✅ NO DATA LOSS
```

### Scenario 2: BREAKING Schema Changes (Dangerous) ⚠️

**v6.1.0 Changes:**
```sql
-- Renamed column (breaking)
EXEC sp_rename 'Orders.CustomerName', 'CustomerFullName', 'COLUMN';
```

**Rollback:**
```bash
# Deploy previous app version
kubectl set image deployment/myapp myapp=myapp:v6.0.0

# App v6.0.0 expects CustomerName column
# Database has CustomerFullName (CustomerName doesn't exist)
# ❌ FAILS: Invalid column name error
```

**Only Solution:** Restore database backup
```sql
RESTORE DATABASE MyAppDB FROM DISK = 'C:\Backup\v6.0.0.bak'
-- ⚠️ DATA LOSS: All data after v6.1.0 deployment lost!
```

### Scenario 3: Forward-Fix (RECOMMENDED) ✅

**Instead of rolling back:**
```bash
# Create hotfix/v6.1.1
# Fix bug in application code
# Keep database at v6.1.0 schema
# Deploy hotfix - compatible with current schema
# ✅ NO DATA LOSS
```

**Golden Rule:** Design for forward-fixes, not rollbacks.

---

## Schema Change Patterns

### Pattern 1: Expand-Contract (Recommended)

**Phase 1 - EXPAND (v6.1.0):**
```sql
-- Add new columns, keep old
ALTER TABLE Orders ADD FirstName NVARCHAR(100) NULL;
ALTER TABLE Orders ADD LastName NVARCHAR(100) NULL;
-- CustomerName still exists (rollback safety)
```

**Application (dual-write):**
```csharp
var order = new Order {
    FirstName = request.FirstName,        // New
    LastName = request.LastName,          // New
    CustomerName = $"{FirstName} {LastName}"  // Old (for rollback)
};
```

**Phase 2 - CONTRACT (v6.2.0, weeks later):**
```sql
-- After stable, remove old column
ALTER TABLE Orders DROP COLUMN CustomerName;
```

### Pattern 2: Backward-Compatible Changes (Always Safe)

```sql
✅ Add nullable column
✅ Add column with default value
✅ Add new table (not used by old code)
✅ Add index (transparent to app)
✅ Add stored procedure (old code doesn't call it)

❌ Drop column (old code expects it)
❌ Rename column (old code uses old name)
❌ Add NOT NULL column without default
```

---

## .dacpac in Branching Strategies

### Current State: Problem

```
dev-6.0.0 → Build .dacpac #1 → DEV
intg      → Build .dacpac #2 → INTG
qa        → Build .dacpac #3 → QA
stage     → Build .dacpac #4 → STAGE
main      → Build .dacpac #5 → PROD

❌ Rebuilt 5 times (not BODE)
❌ Cannot guarantee QA .dacpac == PROD .dacpac
```

### GitFlow: Solution

```
release/v6.1.0 → BUILD ONCE → MyDatabase.dacpac v6.1.0
                              ↓
                        Artifact Registry
                              ↓
                    ├─→ QA (promote)
                    ├─→ STAGE (promote)
                    └─→ PROD (promote)

✅ Build once, promote everywhere
✅ Same artifact tested = deployed
```

### Trunk-Based Development: Enhanced with Feature Flags

```
main → BUILD ONCE → MyDatabase.dacpac
       ↓
   Auto-deploy to all environments
       ↓
   Schema changes applied
   BUT feature flag OFF
       ↓
   Gradually enable flag: 0% → 10% → 100%
```

**Example:**

**Deploy v6.1.0:**
- Schema: Has FirstName, LastName columns (added)
- Feature flag: "use-new-customer-fields" = OFF
- Code: Old path uses CustomerName

**Enable flag gradually:**
- Day 1: 10% users use new columns
- Day 3: 50% users use new columns  
- Day 5: 100% users use new columns

**If issues detected:**
```bash
# Set flag to 0% (30 seconds)
# All users revert to old code path (CustomerName)
# ✅ NO DATABASE ROLLBACK
# ✅ NO DATA LOSS
```

---

## App + .dacpac Versioning

**Critical Rule:** App and database must be versioned together.

```
Release v6.1.0:
├── MyApp.dll (v6.1.0)
└── MyApp.dacpac (v6.1.0)

Both built together, tagged together, promoted together.
```

**Pipeline Example:**
```yaml
build:
  script:
    # Build app
    - dotnet build MyApp.sln -c Release
    # Build .dacpac from SQL project
    - MSBuild MyApp.Database.sqlproj
    # Package together with same version
    - docker build -t myapp:${VERSION}
    - az artifacts publish --name MyApp.dacpac --version ${VERSION}

deploy:
  script:
    # Download SAME version of both
    - docker pull myapp:${VERSION}
    - az artifacts download --name MyApp.dacpac --version ${VERSION}
    # Deploy together
    - kubectl set image deployment/myapp myapp=myapp:${VERSION}
    - SqlPackage.exe /Action:Publish /SourceFile:MyApp.dacpac
```

---

## Best Practices

1. **Always backup before deployment**
   ```bash
   Backup-SqlDatabase -ServerInstance "PROD-SQL" -Database "MyAppDB"
   SqlPackage.exe /Action:Publish /SourceFile:MyApp.dacpac ...
   ```

2. **Use BlockOnPossibleDataLoss**
   ```bash
   SqlPackage.exe /p:BlockOnPossibleDataLoss=true
   # Prevents dropping columns with data, changing data types, etc.
   ```

3. **Make schema changes backward-compatible**
   - Add columns as nullable or with defaults
   - Use Expand-Contract pattern for major changes
   - Avoid dropping/renaming columns

4. **Version .dacpac artifacts**
   ```
   Artifact Registry:
   ├── MyApp.dacpac v6.0.0
   ├── MyApp.dacpac v6.0.1
   └── MyApp.dacpac v6.1.0
   
   Immutable, can deploy any version anytime
   ```

5. **Test in lower environments first**
   ```
   DEV → QA → STAGE → PROD
   (Same artifact promoted, tested at each stage)
   ```

6. **Prefer forward-fixes over rollbacks**
   - Database rollbacks often require backup restore = data loss
   - Forward-fix: Create hotfix with bug fix, keep schema

---

## Summary

### Key Takeaways

- **Conditional BODE:** Same .dacpac artifact, different SQL execution per environment
- **State-based:** Hard to reverse - design for forward-fixes
- **Rollback risk:** Breaking schema changes require backup restore (data loss)
- **Expand-Contract:** Pattern for safe schema evolution
- **Versioning:** App + .dacpac versioned and deployed together
- **Feature flags (TBD):** Decouple schema deployment from feature release

### Migration Path

**Current → GitFlow:**
- Implement .dacpac artifact promotion
- Build once from release branch
- Establish backup procedures

**GitFlow → TBD:**
- Add feature flag capability
- Deploy schema before enabling features
- Enable instant "rollback" via flag disable

The .dacpac pattern is consistent across both approaches - the key difference is TBD's feature flags allow safer deployment by decoupling schema from feature activation.
