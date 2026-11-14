# Morgana — leveraging Mistral Workflows on a 5-seat Mistral Team

Status: draft proposal — 2026-11-14. All user-facing text in the workflows is French; tool and artifact names stay as they are.

## 1. Context

- **Morgana SAS** — Groslay (Val-d'Oise), SIREN 542 042 569, founded 1948, ~30 employees (PME), part of ENEX Technologies. Designs and manufactures finned-pack heat exchangers (evaporators, condensers, hot/cold coils) on demand from its factory north of Paris, for the food, refrigeration and HVAC segments. Three activities: **manufactures to order**, **buys components for resale**, and **provides maintenance services**.
- **Systems**: Sage 100 (gestion commerciale CIAL, comptabilité CPTA, trésorerie TRES, immobilisations IMMO) exposed to Mistral through an MCP connector (POC in this workspace: `sage100_lumiora_04fe`); Microsoft 365 (SharePoint, Outlook shared mailbox, Teams).
- **Team** (5 seats on Mistral Team): 1 accountant, 2 sales admins (quotations, orders, invoices), 1 finance manager, 1 CEO.

## 2. Ground rules inherited from the Sage connector design

1. The Sage MCP connector is **read-only for data**. The only path to a Sage write is: candidate fact → `coord_proposition_enregistrer` (SQL verifies it against Sage) → a person accepts (`coord_proposition_decider`) → `coord_proposition_deposer_demande` → `coord_demande_valider` by an authorized validator (server-side) → the integration program writes through Objets Métiers. **A workflow never writes to Sage directly.**
2. The actor of every decision is the **connected user** (on-behalf-of); the server reads it from the token, checks the validator role, the stale-token `jeton`, and makes replays idempotent. A workflow never passes a user name.
3. **Never compute a business number** (amounts, dates, delays, rankings): every figure is copied verbatim from a tool row. If a column is missing, say so.
4. A document reference always carries its **domain** (purchase vs sales): on real dossiers the same number can exist on both sides (e.g. BC2600013).
5. Customer-facing messages never contain internal `causes` (supplier names, purchase orders).

## 3. Workflow catalog

### W1 — `ma-liste-journaliere` (morning decision desk)
- **Trigger**: schedule daily 08:00 Europe/Paris; also started manually from Le Chat (`+` → Workflows).
- **Steps**: `dossier_contexte` → `coord_ma_liste` → `coord_avis_clients` → read `coord_propositions` (`statut eq 'en_attente'`) → `coord_demandes_a_valider`. Present one item at a time, in French, in the server's order. `wait_for_input` with the choices **Envoyer / Modifier / Fait autrement / Plus tard / Ne pas faire**, and **Accepter / Refuser** (refusal reason via a form) for proposals and demandes. Apply with `coord_decider`, `coord_proposition_decider` + `coord_proposition_deposer_demande`, and `coord_demande_valider(demande, decision, jeton, motif)` passing the `jeton` unchanged (re-read the list on `jeton_perime`). Show the tool's French `message` verbatim; a refusal is an answer, never retried.
- **Who**: sales admins decide supplier chases and customer notices; finance manager / accountant / CEO validate write requests (the server enforces `vous_pouvez_valider`; the author of a request cannot validate it).
- **Output**: `ChatAssistantWorkflowOutput`, published with `available_in_chat_assistant=true`. Note: an equivalent workflow already exists in the workspace (`demandes-a-valider`, deployment `lumiora-validation-essai`, hardened, on-behalf-of) — extend it rather than duplicate it.

### W2 — `relance-fournisseur` (supplier replies become checked facts)
- **Trigger**: webhook on new mail in the shared mailbox (paid tier — on the free plan triggers answer 403; fallback: hourly schedule), or a Teams/Slack message.
- **Steps**: the model may only *read* the message and extract candidate facts (quote, dates, document + domain); then `coord_interaction_enregistrer(entrant, ...)` and `coord_proposition_enregistrer(origine=message, nature, piece, domaine, tiers, citation, id_interaction)` — SQL verifies the fact (`gescom_verifier_fait_commande` rules). If `enregistre`, notify the decider (Teams) and `wait_for_input` Accepter/Refuser; on accept → `coord_proposition_decider(accepter)` + `coord_proposition_deposer_demande`.
- **Who**: sales admins.

### W3 — `avis-client-retard` (customer delay notices)
- **Trigger**: schedule daily 09:00; re-surfaces automatically on `faits_changes`.
- **Steps**: `gescom_avis_clients` / `coord_avis_clients` (server order, never re-sort, never announce an invented date — `date_a_confirmer` means warn without a date). Draft via `coord_brouillon_enregistrer` (customer-facing text only). Human picks **Envoyer** → link the message before sending (`coord_brouillon_lier_message`) → send from the shared mailbox (Graph, immutable ids, one delta link per folder) → `coord_interaction_enregistrer(sortant, envoye, ref_message)` (replayable by `ref_message`; subject only, never the body). A client reply → `gescom_verifier_reponse_client` → W2 path.
- **Who**: sales admins.

### W4 — `preparation-facture` (order → Sage-ready invoice file)
- **Trigger**: new order email in the shared mailbox, a SharePoint list item, a Teams message, or manually from Le Chat.
- **Steps**: resolve the customer's `compte tiers` and articles (`describe_entities`, `read_records`); check stock/coverage (`gescom_stock`, `gescom_lignes_client_a_risque`, `gescom_ruptures_annoncees`); build the invoice file (CSV `Date;Compte tiers;Montant eur`, the pattern of the existing order agent) or a structured draft; save to a SharePoint library and the Mistral library; `wait_for_input` confirmation by the sales admin. **Posting into Sage is not done by the workflow**: it goes through the demande circuit with finance-manager validation. The same flow covers **maintenance services** (intervention report in SharePoint → service invoice lines).
- **Who**: sales admins prepare; finance manager validates.

### W5 — `rapport-hebdo-tresorerie` (weekly finance report)
- **Trigger**: schedule Mondays 08:00. Unattended runs cannot be on-behalf-of, so this needs a **workspace/org-level credential** on the Sage connector (or is started by the finance manager).
- **Steps**: `compta_previsionnel_tresorerie` (+ `_detail` for weeks that need explaining; cite `perimetre_exclu` and the `certitude` column — never mix comptable and estimé), `compta_balance_agee` + `compta_pieces_dues`, `compta_besoin_fonds_roulement`, `compta_tva_exigible`, `gescom_ventes` (grouped by month / family / client), `gescom_ruptures_annoncees`, `gescom_retard_fournisseur`.
- **Output**: French Markdown report → Teams channel + SharePoint + email to CEO and finance manager. Read-only, no human gate.
- **Who**: finance manager reviews; CEO receives.

### W6 — `cloture-mensuelle` (month-end checklist)
- **Trigger**: schedule last business day 18:00.
- **Steps**: `compta_controles`; `compta_journal` (AN, ACH, VTE, OD); `compta_balance` (70, 401, 411, 445…); `compta_tva_exigible`; grand-livre spot checks (`compta_grand_livre`). Produce a checklist document in SharePoint; `wait_for_input` sign-off by the accountant; anomalies → Teams card to the finance manager.
- **Who**: accountant.

### W7 — `tableau-de-bord` (CEO dashboard, chat)
- **Trigger**: manual, from Le Chat (chat-assistant workflow).
- **Steps**: answer in French from tool rows only — `gescom_ventes` (axes: month, exercice, family, client, article), `compta_balance_par_exercice`, `compta_balance_agee`, `gescom_stock_projection`, `gescom_ruptures_annoncees`, `gescom_retard_fournisseur`, plus row counts of `coord_ma_liste` / `coord_avis_clients`.
- **Who**: CEO (and finance manager).

### W8 — `alertes-appro` (purchasing / stock alerts)
- **Trigger**: schedule daily 09:00.
- **Steps**: `gescom_ruptures_annoncees`, `gescom_stock_projection`, `gescom_lignes_client_a_risque`, `gescom_qualite_donnees_orchestration`, `gescom_relances_fournisseurs`; digest posted to the Teams purchasing channel; items needing a decision surface in W1.
- **Who**: sales admins / buyers.

## 4. Role × workflow matrix

| Role (1 seat each unless noted) | W1 desk | W2 supplier replies | W3 customer notices | W4 invoices | W5 weekly report | W6 closing | W7 dashboard | W8 alerts |
|---|---|---|---|---|---|---|---|---|
| Sales admin ×2 | decides | decides | decides | prepares | — | — | — | watches |
| Accountant ×1 | validates demandes | — | — | — | — | runs + signs off | — | — |
| Finance manager ×1 | validates demandes | approves proposals | approves | validates | owns report | receives anomalies | uses | watches |
| CEO ×1 | validates exceptions | — | — | — | receives report | — | uses | — |

## 5. Implementation notes (Mistral Studio)

- Python Workflows SDK (`mistralai-workflows`), one Git repo, **managed deployments** (`deploy_workflow_deployment`); set `hardened=true` for any on-behalf-of workflow.
- **Schedules**: `create_workflow_schedule` (cron, Europe/Paris). **Webhook triggers** (`create_trigger`) require a paid tier — on the current free plan they answer 403; confirm they are included in the Team plan.
- Human-in-the-loop: `wait_for_input` with `ConfirmationInput` / `FormInput`; external approval via `signal_workflow_execution` (`approve` / `resume`) and `query_workflow_execution`. Set `execution_timeout` explicitly — the SDK default (1 h) kills an approval left overnight (e.g. `timedelta(days=7)`).
- Connectors in code use the **Studio name** (`sage100_lumiora_04fe`), `run_as="auto"` + `on_behalf_of=True` for decision workflows; keep a single connector exposing a given tool set (two connectors with the same tools confuse the chat model).
- Publish chat workflows so they appear in Le Chat: entrypoint annotated `-> ChatAssistantWorkflowOutput`, then `PUT /v1/workflows/{id} {"available_in_chat_assistant": true}`.
- MCP transport detail: the DAB-backed server wraps rows twice (`{"value":{"value":[...]}}`) — unwrap in code and test against **one real server answer**, not only fakes.
- Privacy / ops: payload encryption on, `OTEL_ENABLED=false`, no business text in workflow inputs or search keys, secrets only in the workspace secrets manager (never in code, logs or chat). The EU endpoint and zero-retention cover **stateless model calls only**; Workflows history and Vibe Work sit at Mistral — get the client's written consent (data residency) before go-live, and get execution/schedule pricing in writing (not published).
- M365: SharePoint libraries for reports / CSV / checklists; shared mailbox through a dedicated Entra app (delegated Graph, immutable ids, per-folder delta links, delta links treated as sensitive state); Teams for notifications.

## 6. Phased rollout

- **P0 (week 1)**: W1 decision desk (extend the existing `demandes-a-valider` deployment) + the `ma-liste` skill shared with the team; W7 dashboard.
- **P1 (weeks 2–3)**: W2 supplier chases, W3 customer delay notices.
- **P2 (month 2)**: W4 invoice preparation (incl. maintenance interventions), W8 alerts.
- **P3 (months 2–3)**: W5 weekly treasury report, W6 month-end closing.

## 7. Prerequisites / open questions for Morgana

1. Mistral Team plan: 5 seats, confirm triggers/webhooks included, pricing in writing for executions, schedules and history retention.
2. A Sage 100 MCP connector **per company dossier** — the POC connector points at the LUMIORA dossier; Morgana needs its own deployment of the same MCP server (plus the coordination SQL module if write-requests are wanted).
3. A workspace/org-level credential on the Sage connector for unattended schedules (schedules cannot run on-behalf-of), or accept that scheduled runs are started by a person.
4. M365: SharePoint site + libraries, shared mailbox + Entra app registration with admin consent, Teams channels.
5. Validator rules in Sage for `demande_*` (finance manager + accountant; CEO for exceptions), including "the author cannot validate their own request".
6. Data-residency consent for Vibe Work / Workflows history at Mistral.
