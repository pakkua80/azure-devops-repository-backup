# Bugs identifiés - Azure DevOps Repository Backup

**Date**: 2025-10-23
**Analysé par**: Claude Code
**Version analysée**: 1.0.2

---

## 🔴 Bugs critiques

### Bug #1: Variable POSITIONAL non initialisée
**Fichier**: `src/backup-devops.sh`
**Ligne**: 123
**Sévérité**: 🔴 Critique

**Description**:
Le script utilise une variable tableau `POSITIONAL` qui n'est jamais initialisée, provoquant une erreur "unbound variable" à cause du mode strict (`set -u`).

**Code problématique**:
```bash
set -- "${POSITIONAL[@]}" # restore positional parameters
```

**Erreur produite**:
```
bash: POSITIONAL: unbound variable
```

**Impact**: Le script crashe systématiquement à cette ligne.

**Solution proposée**: Supprimer la ligne 123 car le script n'utilise pas de paramètres positionnels après le parsing des options.

---

### Bug #2: Appel de fonction avant sa définition
**Fichier**: `src/backup-devops.sh`
**Ligne**: 10
**Sévérité**: 🔴 Critique

**Description**:
La fonction `die` est appelée ligne 10 mais définie lignes 37-40.

**Code problématique**:
```bash
[[ "${BASH_VERSINFO[0]}" -lt 4 ]] && die "Bash >=4 required"
```

**Impact**: Erreur "command not found" si Bash version < 4.

**Solution proposée**: Déplacer la définition de la fonction `die` avant la ligne 10, ou déplacer le check de version après les définitions de fonctions.

---

### Bug #3: Extension tar incorrecte
**Fichier**: `src/backup-devops.sh`
**Lignes**: 272, 291, 295, 297
**Sévérité**: 🔴 Critique

**Description**:
Le script crée des archives avec l'extension `.tar.bz` (ligne 272) mais la politique de rétention cherche des fichiers `*.tar.bz` (lignes 291, 295, 297).

**Code problématique**:
```bash
# Ligne 272 - Création de l'archive
tar cjf ${BACKUP_FOLDER}.tar.bz ${BACKUP_FOLDER}

# Lignes 291, 295, 297 - Recherche pour rétention
find ${BACKUP_ROOT_PATH} -mindepth 1 -maxdepth 1 -type f -name "*.tar.bz" | wc -l
```

**Impact**: La politique de rétention ne trouve jamais les anciens backups à supprimer, causant une accumulation infinie de fichiers de backup.

**Solution proposée**: Utiliser l'extension standard `.tar.bz2` partout.

---

## ⚠️ Bugs moyens

### Bug #4: Option -w ne permet pas de désactiver le wiki
**Fichier**: `src/backup-devops.sh`
**Lignes**: 17, 82
**Sévérité**: ⚠️ Moyen

**Description**:
`PROJECT_WIKI` est déjà `true` par défaut. L'option `-w` le met toujours à `true`, il n'y a aucun moyen de le désactiver.

**Code problématique**:
```bash
# Ligne 17 - Défaut
PROJECT_WIKI=true;

# Ligne 82 - Option -w
w) PROJECT_WIKI=true
   ;;
```

**Impact**: Impossible de désactiver le backup des wikis.

**Solution proposée**:
- Changer le défaut à `false`
- Ou utiliser un flag booléen inversé (ex: `-n` pour "no wiki")

---

### Bug #5: Variables non quotées dans backup-devops.sh
**Fichier**: `src/backup-devops.sh`
**Lignes**: 148, 197, 270, 272, etc.
**Sévérité**: ⚠️ Moyen

**Description**:
De nombreuses variables ne sont pas quotées, ce qui peut causer des problèmes avec des espaces ou caractères spéciaux.

**Exemples problématiques**:
```bash
az devops project list --organization ${ORGANIZATION}  # Ligne 148
cd ${BACKUP_ROOT_PATH}                                  # Ligne 270
tar cjf ${BACKUP_FOLDER}.tar.bz ${BACKUP_FOLDER}       # Ligne 272
```

**Impact**: Échec du script si les valeurs contiennent des caractères spéciaux ou espaces.

**Solution proposée**: Quoter toutes les variables: `"${ORGANIZATION}"`, `"${BACKUP_ROOT_PATH}"`, etc.

---

### Bug #6: Utilisation dangereuse de eval
**Fichier**: `src/backup-devops.sh`
**Ligne**: 282
**Sévérité**: ⚠️ Moyen (Sécurité)

**Description**:
Utilisation de `eval` qui n'est pas nécessaire et constitue un risque de sécurité.

**Code problématique**:
```bash
eval "echo Elapsed time : $(date -ud "@$elapsed" +'$((%s/3600/24)) days %H hr %M min %S sec')"
```

**Impact**: Vulnérabilité potentielle d'injection de commande.

**Solution proposée**: Supprimer `eval` et utiliser directement `echo`.

---

## 🔴 Bugs critiques dans docker-script.sh

### Bug #7: Option incorrecte pour dry-run
**Fichier**: `src/docker-script.sh`
**Lignes**: 7, 10
**Sévérité**: 🔴 Critique

**Description**:
Le script utilise `--dryrun true` alors que `backup-devops.sh` attend l'option `-x` pour activer le mode dry-run.

**Code problématique**:
```bash
./backup-devops.sh -p $DEVOPS_PAT -o $DEVOPS_ORG_URL -d /data -r $RETENTION_DAYS --dryrun true
```

**Impact**: Le mode dry-run ne fonctionne jamais dans Docker.

**Solution proposée**: Remplacer `--dryrun true` par `-x`.

---

### Bug #8: Variables non quotées dans docker-script.sh
**Fichier**: `src/docker-script.sh`
**Lignes**: 7, 10, 14, 17
**Sévérité**: ⚠️ Moyen

**Description**:
Toutes les variables devraient être quotées pour éviter les problèmes avec des espaces.

**Code problématique**:
```bash
./backup-devops.sh -p $DEVOPS_PAT -o $DEVOPS_ORG_URL -d /data -r $RETENTION_DAYS
```

**Impact**: Échec si les valeurs contiennent des espaces.

**Solution proposée**: Quoter toutes les variables: `"$DEVOPS_PAT"`, `"$DEVOPS_ORG_URL"`, etc.

---

## 📊 Résumé

| Sévérité | Nombre | Description |
|----------|--------|-------------|
| 🔴 Critique | 4 | Bugs qui empêchent le fonctionnement du script |
| ⚠️ Moyen | 4 | Bugs qui causent des problèmes dans certains cas |
| **Total** | **8** | **Bugs identifiés** |

---

## 🔧 Ordre de correction recommandé

1. **Bug #1** - Variable POSITIONAL (crash immédiat)
2. **Bug #7** - Option dry-run Docker (fonctionnalité cassée)
3. **Bug #3** - Extension tar (rétention cassée)
4. **Bug #2** - Fonction die (crash potentiel)
5. **Bug #5 & #8** - Variables non quotées (robustesse)
6. **Bug #4** - Option wiki (amélioration UX)
7. **Bug #6** - Eval (sécurité)

---

**Note**: Ce rapport a été généré par analyse statique du code. Des tests unitaires sont recommandés pour confirmer chaque bug.
