# Réponses du test

## _Utilisation de la solution (étape 1 à 3)_

# ========== Documentation technique ==========
### Étape 1
But du projet : Construire un pipeline simple de collecte de données utilisateurs, structurer les profils, assurer la qualité des données, et valider le tout par des tests unitaires.

Fichiers principaux : 
test_pipeline.ipynb : fonctions pour recupérér les données puis analyse exploratoire des données, contrôles qualité et tests unitaires.

requirements.txt : liste des bibliothèques à installer.

### Étape 2
import requests
import pandas as pd
import unittest

BASE_URL = "http://127.0.0.1:8000"

# ========== Fetch data ==========
def fetch_users():
    return requests.get(f"{BASE_URL}/users").json()
    
def fetch_tracks():
    return requests.get(f"{BASE_URL}/tracks").json()
    
def fetch_listening_history():
    return requests.get(f"{BASE_URL}/listen_history").json()  

# ========== Data exploration ==========
users_data = fetch_users()
user_items = users_data["items"]
df_users   = pd.DataFrame(user_items)
df_users.head(5)

listening_data = fetch_listening_history()
history_items  = listening_data["items"]
df_history     = pd.DataFrame(history_items)
df_history.head(5)

# Flattening listening history data
History_flattened = []

for _, row in df_history.iterrows():
    user_id = row["user_id"]  
    for song_id in row["items"]:
        History_flattened.append({
            "user_id": user_id,
            "song_id": song_id,
        })
df_history_flat = pd.DataFrame(History_flattened)
print(df_history_flat.sample(5))

### Étape 3

# ========== Unit Test ==========
# ========== Test 1: API returns a response (status 200) ==========
try:
    response = requests.get(f"{BASE_URL}/users")
    assert response.status_code == 200, "API /users not reachable"
    print("Test 1 passed: API /users is reachable")
except Exception as e:
    print("Test 1 failed:", e)

# ========== Test 2: Response is a JSON with expected structure ==========
try:
    users_data = fetch_users()
    assert isinstance(users_data, dict), "/users response is not a dictionary"
    assert "items" in users_data, "Key 'items' not found in /users response"
    assert isinstance(users_data["items"], list), "'items' is not a list"
    print("Test 2 passed: /users returns a valid JSON structure")
except Exception as e:
    print("Test 2 failed:", e)

# ========== Test 3: Required fields exist in first user ==========
try:
    user = users_data["items"][0]
    assert "id" in user, "'id' missing in user"
    assert "email" in user, "'email' missing in user"
    assert isinstance(user["id"], int), "'id' is not an integer"
    assert isinstance(user["email"], str), "'email' is not a string"
    print("Test 3 passed: User contains required fields with correct types")
except Exception as e:
    print("Test 3 failed:", e)

# ========== Test 4: Listening history is valid ==========
try:
    history_data = fetch_listening_history()
    assert "items" in history_data, "Key 'items' missing in /listen_history"
    assert isinstance(history_data["items"], list), "Listening history items not a list"
    print("Test 4 passed: Listening history structure is valid")
except Exception as e:
    print("Test 4 failed:", e)

# ========== Test 5: Songs endpoint is working and structured ==========
try:
    songs_data = fetch_songs()
    assert isinstance(songs_data, dict), "Songs response not a dictionary"
    assert "items" in songs_data, "Key 'items' not in /songs"
    assert isinstance(songs_data["items"], list), "Songs 'items' is not a list"
    print("Test 5 passed: Songs endpoint returns valid structure")
except Exception as e:
    print("Test 5 failed:", e)


# ========== Data quality ==========

# Checking for missing values 
missing_df = df_users.isnull().sum().reset_index()
missing_df.columns = ['Column', 'Missing Values']
missing_df.head(5)

# Checking for duplicate user IDs
duplicate_ids = df_users["id"].duplicated().sum()
print(f"Number of duplicate user IDs: {duplicate_ids}")

# Check for users with no listening history
users_no_history = df_history[df_history["items"].apply(lambda x: len(x) == 0)]
no_history_count = len(users_no_history)

print(f"Number of users without listening history: {no_history_count}")
if no_history_count > 0:
    display(users_no_history.head())

# Checking for users whose creation date is later than their last update date 
anomaly_df = df_users[df_users["created_at"] > df_users["updated_at"]]
anomaly_count = len(anomaly_df)
print(f"Number of users with a creation date later than the update date: {anomaly_count}")
if anomaly_count > 0:
    display(anomaly_df.head())

# ========== Validate Songs ==========
try:
    songs_data = fetch_songs()
    items = songs_data.get("items", [])
    
    assert isinstance(items, list), "'items' should be a list"

    for i, song in enumerate(items):
        assert "id" in song, f"Missing 'id' in item {i}"
        assert isinstance(song["id"], int), f"'id' is not an int in item {i}"

        assert "name" in song, f"Missing 'name' in item {i}"
        assert isinstance(song["name"], str), f"'name' is not a string in item {i}"
        assert song["name"].strip() != "", f"'name' is empty in item {i}"
    
    print("Songs have valid 'id' and 'name'")
except AssertionError as e:
    print(f"Data quality test failed: {e}")
except Exception as e:
    print(f"Unexpected error during data quality test: {e}")


# ========== Validate Listening History Entries ==========
try:
    history_data = fetch_listening_history()
    items = history_data.get("items", [])
    
    assert isinstance(items, list), "Expected 'items' to be a list"

    for i, entry in enumerate(items):
        assert "user_id" in entry, f"Missing 'user_id' in item {i}"
        assert isinstance(entry["user_id"], int), f"'user_id' is not an int in item {i}"
        
        assert "items" in entry, f"Missing 'items' (song list) in item {i}"
        assert isinstance(entry["items"], list), f"'items' is not a list in item {i}"
        
        assert len(entry["items"]) > 0, f"'items' list is empty in item {i}"

        assert "created_at" in entry, f"Missing 'created_at' in item {i}"
        assert "updated_at" in entry, f"Missing 'updated_at' in item {i}"
    
    print("Listening history entries are valid")
except AssertionError as e:
    print(f"Data quality test failed: {e}")
except Exception as e:
    print(f"Unexpected error during data quality test: {e}")

## Questions (étapes 4 à 7)

### Étape 4

# ========== SCHEMA ==========

# USERS :
- id              BIGINT PRIMARY KEY
- first_name      VARCHAR
- last_name       VARCHAR
- email           VARCHAR
- gender          VARCHAR
- favorite_genres VARCHAR
- created_at      TIMESTAMP
- updated_at      TIMESTAMP

# SONGS : 
- id           BIGINT PRIMARY KEY
- name         VARCHAR
- artist       VARCHAR
- songwriters  VARCHAR
- duration     TIME
- genres       VARCHAR
- album        VARCHAR
- created_at   TIMESTAMP
- updated_at   TIMESTAMP

# LISTENING_HISTORY
- id           BIGINT PRIMARY KEY
- user_id      BIGINT REFERENCES users(id)
- song_id      BIGINT REFERENCES songs(id)

# ========== SGBD ========== 

Azure Database for PostgreSQL - Hyperscale (Citus) 
Avantages : 
 - Scalabilité horizontale :Permet de gérer une grande volumétrie de données ou de requêtes en ajoutant plusieurs serveurs (nœuds) au lieu de dépendre uniquement des ressources d’un seul serveur.
 - Adapté aux tables relationnelles volumineuses : Grâce à sa capacité de sharding (répartition automatique des données), il est idéal pour des tables contenant potentiellement des millions de lignes.
 - Compatible avec les outils BI (Power bi par exemple)
 - Indexation performance : Supporte différents types d’index (B-tree, GIN, etc.) pour garantir des accès rapides aux données, même à très grande échelle.

# ========== Architecture complète==========

     [ FastAPI ]
          |
          v
   [ Azure Databricks ]
     (Script Python)
          |
          v
 [ Azure Blob Storage ]
     (Landing Zone)
          |
          v
[ Azure Data Factory ]
(ETL + Automatisation quotidienne)
          |
          v
[ Azure PostgreSQL - Hyperscale ]
(Stockage principal)
   
### Étape 5
# ========== Monitoring==========
1- Databricks
   - Statut des jobs
   - Durée d'exécution
   - Logs d'erreur (lier databricks avec Azure Monitor ou log analytics)
   - Envoi d'alertes via email en cas d'échoue
   - 
2- Data factory 
   - Utilisation des alertes ADF pour notifier les erreurs
   - Visualisation des logs via Azure Monitor
     
3- Azure PostegreSQL
   - Supervision du serveur (CPU, mémoire)
   - Intégrer avec Azure monitor pour générer des alertes en cas de surcharge

### Étape 6
# ========== Automatisation du calcul des recommandations ========== 
1- Données en provenance de Azure PostgreSQL
2- Job databricks pour charger les données -> chargement du modèle existant -> stockage des recommandations dans postgreSQL)
3- Déclenchement du traitement du job databricks via Data factory
### Étape 7
# ========== Automatisation du réentrainement du modèle de recommandation ========== 
1- Job databricks pour récupérer les échantillon de données nécessaires à partir de postgreSQL et un autre job pour entrainer le modèle fait par le data scientist
2- Test et validation automatique (à definir)
3- Déclenchement hebdomadaire (via Datafactory) automatique ou lorsqu'on a un nombre précis de nouvelles données (à définir)

