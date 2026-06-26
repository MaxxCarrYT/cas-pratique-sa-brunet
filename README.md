# Automatisations n8n - Cas pratique SA Brunet

Deux automatisations construites sur **n8n** pour répondre à un cas pratique : réduire la saisie manuelle des factures fournisseurs et automatiser les relances clients d'une PME industrielle agroalimentaire (ERP de type Odoo).

Auteur : **Maxime Carrière**

> Ce dépôt est fourni à titre de démonstration dans le cadre d'une candidature. Voir la note en bas de page sur l'usage et la propriété.

---

## Ce que contient le dépôt

- `facture-fournisseur.json` — workflow de saisie assistée des factures fournisseurs
- `relance-client.json` — workflow de relance automatique des factures clients impayées
- `README.md` — ce fichier

Les deux fichiers sont des exports n8n importables tels quels (menu *Import from File*). Ce sont des versions de démonstration : données fictives, seuils visibles, aucun identifiant réel.

---

## Workflow 1 — Facture fournisseur

Objectif : transformer une facture PDF reçue par mail en écriture comptable, sans saisie manuelle, et sans jamais écrire dans l'ERP sans validation humaine quand un doute existe.

Déclencheurs : un planificateur et une lecture des mails non lus (IMAP).

Le flux récupère le PDF, en extrait le texte, le fait lire par un modèle qui renvoie un JSON structuré (fournisseur, numéro, montants HT/TVA/TTC, date, catégorie, indice de confiance). Un nœud de contrôle applique ensuite cinq garde-fous avant toute écriture.

Les garde-fous :

1. Cohérence des montants. Le TTC doit valoir HT + TVA, à 0,02 € près.
2. Champs obligatoires présents (fournisseur, numéro, date, TTC).
3. Détection de doublon contre les factures déjà comptabilisées.
4. Seuil de montant élevé (10 000 €), qui déclenche une validation renforcée sans bloquer.
5. Seuil de confiance d'extraction minimum (0,6).

Si tout passe, l'écriture part vers le nœud Odoo (`account.move`). Si quelque chose bloque, un mail interne ouvre une console de correction et le traitement se met en pause jusqu'à 48 h. Sur cette console, la personne au service comptabilité choisit entre deux actions : corriger elle-même les champs, ou renvoyer la facture au fournisseur d'origine pour qu'il la corrige. Dans ce second cas, un mail part automatiquement à l'expéditeur de la facture en reprenant les anomalies détectées, et aucune écriture comptable n'a lieu.

Le workflow a donc trois issues possibles, toutes tracées dans un journal d'audit (n8n Data Table) avec horodatage et mention RGPD : facture corrigée puis validée (écriture Odoo), facture renvoyée au fournisseur pour correction, ou traitement abandonné faute de réponse sous 48 h.

Note sur le nœud Odoo : il est présent dans le flux et c'est lui qui crée l'écriture, mais il est laissé sans identifiant de connexion (pas de credential Odoo de test). Les seuils et la liste des doublons sont simulés dans le code et commentés pour pointer où brancher une vraie requête ERP en production.

## Workflow 2 - Relance client

Objectif : relancer les factures impayées tous les matins, avec un ton qui s'adapte au retard, et passer la main à un humain quand le dossier devient sensible.

Déclencheur : tous les jours à 9 h.

Le flux lit la base des factures (Airtable dans cette démo, l'ERP en production), calcule pour chacune le retard et le niveau de relance (première à 30 jours, suivantes tous les 10 jours, ton qui durcit du courtois à la mise en demeure), rédige le mail adapté, vérifie qu'il est complet avant l'envoi, met à jour la base, et marque une pause anti-spam entre deux envois. Au-delà d'un certain niveau ou d'un certain retard, une alerte interne escalade le dossier vers le service recouvrement.

Les garde-fous : une facture payée n'est jamais relancée, l'IA n'invente aucune donnée et a interdiction de chiffrer la moindre pénalité (c'est la compta qui calcule), le ton reste légalement correct même en mise en demeure, et l'envoi est bloqué si un champ requis manque.

---

## Stack technique

- **n8n** comme orchestrateur (auto-hébergeable, donc les données peuvent rester en interne)
- Modèle de langage via OpenRouter pour l'extraction et la rédaction
- Airtable comme base clients de démonstration (remplaçable par l'ERP)
- n8n Data Table comme journal d'audit

Choix d'IA : Claude Haiku a été utilisé en développement pour aller vite. En production, le modèle tournerait **en local** pour que les données comptables ne quittent pas l'entreprise.

---

## Importer les workflows dans n8n

1. Dans n8n, ouvrir le menu des workflows puis *Import from File*.
2. Sélectionner `facture-fournisseur.json` ou `relance-client.json`.
3. Rebrancher vos propres credentials (mail, OpenRouter ou modèle local, Airtable ou Odoo).
4. Ajuster les seuils des garde-fous dans les nœuds Code selon vos volumes réels.

---

## Usage et propriété

Code et workflows réalisés par Maxime Carrière, fournis pour évaluation dans le cadre d'une candidature. Tous droits réservés. Réutilisation en production ou diffusion sans accord préalable de l'auteur non autorisée.
