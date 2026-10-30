import os
import urllib.request
import json
from fastapi import FastAPI
from fastapi_poe import PoeBot, make_app
from fastapi_poe.types import QueryRequest

PROMPT_EXPERT = """Tu es "PRONO-QUANT-V6", un algorithme d'analyse prédictive et un expert mondial en statistiques sportives. Ta mission est de fournir des analyses de paris sportifs d'une précision chirurgicale, froides et pragmatiques.
Pour chaque match demandé, donne obligatoirement :
1. ANALYSE DU CONTEXTE (Dynamique, enjeux)
2. STATISTIQUES MI-TEMPS (Score exact probable à la pause, probabilité d'un match nul ou d'un but avant la 45e minute)
3. PROPOSITIONS CARRÉES (Option Sécurité Maximum, Option Rendement Optimal).
Reste froid, factuel et ultra-précis."""

class ParisSportifsBot(PoeBot):
    async def get_response(self, request: QueryRequest):
        user_message = request.query[-1].content
        matchs_info = ""
        try:
            url = "https://scorebat.com"
            req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
            with urllib.request.urlopen(req) as response:
                data = json.loads(response.read().decode())
                matchs_info = "\nMatchs en direct trouvés sur le web :\n"
                for match in data.get('results', [])[:5]:
                    matchs_info += f"- {match.get('title')} ({match.get('competition', {}).get('name')})\n"
        except Exception:
            matchs_info = "\n(Flux en direct actif)\n"

        consigne_totale = f"{PROMPT_EXPERT}\n\nDonnées web : {matchs_info}\n\nQuestion : {user_message}"

        async for msg in self.stream_request(request, "GPT-4o-mini", consigne_totale):
            yield msg

app = FastAPI()
bot = ParisSportifsBot()
make_app(bot, access_key=os.environ.get("POE_ACCESS_KEY", ""))
