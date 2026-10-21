import os
import json
import requests
from datetime import datetime, timezone
from google.cloud import firestore
from google.oauth2 import service_account

# ==========================================
# 1. Configuração e Conexão Firestore
# ==========================================
ARQUIVO_CHAVE = "firebase_key.json"
DATABASE_ID = os.getenv("DATABASE_ID", "ai-studio-59483fc8-da5a-456d-a48b-6ca587e00652")

# Autenticação híbrida: usa chave local caso exista ou identidade nativa no Cloud Run
if os.path.exists(ARQUIVO_CHAVE):
    with open(ARQUIVO_CHAVE, "r") as f:
        key_data = json.load(f)
        project_id = key_data.get("project_id")
    credenciais = service_account.Credentials.from_service_account_file(ARQUIVO_CHAVE)
    db = firestore.Client(project=project_id, credentials=credenciais, database=DATABASE_ID)
    print(f"[LOCAL] Conectado via Service Account ao Firestore ({DATABASE_ID})")
else:
    project_id = os.getenv("GOOGLE_CLOUD_PROJECT", "app-robmar-automoveis")
    db = firestore.Client(project=project_id, database=DATABASE_ID)
    print(f"[CLOUD] Conectado via GCP Native Identity ao Firestore ({DATABASE_ID})")

BASE_URL = "https://integracao.autocerto.com"
TOKEN_URL = f"{BASE_URL}/oauth/token"
URL_ESTOQUE = f"{BASE_URL}/api/Veiculo/ObterEstoque"

# ==========================================
# 2. Funções Utilitárias
# ==========================================
def obter_token_autocerto(usuario, senha):
    """Autentica na AutoCerto e retorna o token de acesso."""
    resp = requests.post(
        TOKEN_URL,
        data={"grant_type": "password", "username": usuario, "password": senha},
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        timeout=30
    )
    resp.raise_for_status()
    return resp.json()["access_token"]

def buscar_estoque_autocerto(token, unidade_id):
    """Obtém a lista de veículos em estoque da unidade."""
    headers = {"Authorization": f"Bearer {token}", "Accept": "application/json"}
    params = {"codigoUnidade": int(unidade_id)}
    resp = requests.get(URL_ESTOQUE, headers=headers, params=params, timeout=120)
    resp.raise_for_status()
    return resp.json()

def montar_documento_veiculo(v, loja_id, loja_nome, status="ATIVO", ativo=True, acao_sync="INSERIDO"):
    """Estrutura o documento para salvar na coleção 'estoque'."""
    fotos = v.get("Fotos") or []
    fotos_ord = sorted(fotos, key=lambda x: x.get("Posicao", 999))
    foto_capa = fotos_ord[0].get("URL") if fotos_ord else ""
    
    fotos_formatadas = [
        {
            "codigo": f.get("Codigo"),
            "posicao": f.get("Posicao"),
            "url": f.get("URL")
        }
        for f in fotos_ord if f.get("URL")
    ]
    
    opcionais = v.get("Opcionais") or []
    opcionais_lista = [op.get("Descricao") for op in opcionais if op.get("Descricao")]

    agora_iso = datetime.now(timezone.utc).isoformat()

    return {
        "codigo": int(v.get("Codigo")),
        "placa": str(v.get("Placa") or "").strip().upper(),
        "marca": str(v.get("Marca") or "").strip().upper(),
        "modelo": str(v.get("Modelo") or "").strip().upper(),
        "versao": str(v.get("Versao") or "").strip(),
        "ano_fabricacao": int(v.get("AnoFabricacao") or 0),
        "ano_modelo": int(v.get("AnoModelo") or 0),
        "preco": float(v.get("Preco") or 0.0),
        "preco_classificados": float(v.get("PrecoClassificados") or 0.0),
        "km": float(v.get("Km") or 0.0),
        "cambio": str(v.get("Cambio") or "").strip(),
        "combustivel": str(v.get("Combustivel") or "").strip(),
        "cor": str(v.get("Cor") or "").strip(),
        "categoria": str(v.get("Categoria") or "").strip(),
        "portas": int(v.get("Portas") or 0),
        "zero_km": bool(v.get("ZeroKm")),
        "destaque": bool(v.get("Destaque")),
        "chassi": str(v.get("Chassi") or "").strip(),
        "renavam": str(v.get("Renavam") or "").strip(),
        "url_video": str(v.get("UrlVideo") or "").strip(),
        "observacao": str(v.get("Observacao") or "").strip(),
        "opcionais": opcionais_lista,
        "foto_capa": foto_capa,
        "fotos": fotos_formatadas,
        "qtd_fotos": len(fotos_formatadas),
        "unidade_id": loja_id,
        "unidade_nome": loja_nome,
        "status": status,
        "ativo": ativo,
        "ultima_acao_sync": acao_sync,
        "data_cadastro_api": str(v.get("DataCadastro") or ""),
        "data_modificacao_api": str(v.get("DataModificacao") or ""),
        "updated_at": agora_iso
    }

# ==========================================
# 3. Execução do Job Multi-Loja
# ==========================================
def main():
    print("\nBuscando lojas cadastradas no Firestore...")
    lojas_docs = db.collection("loja").stream()

    for doc_loja in lojas_docs:
        loja_id = doc_loja.id
        dados_loja = doc_loja.to_dict()
        loja_nome = dados_loja.get("name", loja_id)
        loja_ativa = dados_loja.get("active", False)
        
        if not loja_ativa:
            print(f"\n[-] Loja {loja_nome} ({loja_id}) está inativa no sistema. Pulando...")
            continue
        
        # Consulta a subcoleção de credenciais
        cred_ref = db.collection("loja").document(loja_id).collection("credenciais_estoque").document("autocerto")
        cred_doc = cred_ref.get()
        
        if not cred_doc.exists:
            print(f"\n[-] Loja {loja_nome} ({loja_id}): Sem credenciais da AutoCerto cadastradas. Pulando...")
            continue
            
        dados_cred = cred_doc.to_dict()
        if not dados_cred.get("autocerto_ativo", False):
            print(f"\n[-] Loja {loja_nome} ({loja_id}): Integração AutoCerto desativada. Pulando...")
            continue
            
        usuario = dados_cred.get("autocerto_usuario")
        senha = dados_cred.get("autocerto_senha")
        unidade_id = dados_cred.get("autocerto_unidade_id")
        
        if not (usuario and senha and unidade_id):
            print(f"\n[!] Loja {loja_nome} ({loja_id}): Dados de credenciais incompletos. Pulando...")
            continue

        print("\n" + "="*70)
        print(f" SINCRONIZANDO: {loja_nome} (ID: {loja_id} | Unidade AutoCerto: {unidade_id})")
        print("="*70)

        try:
            # 1. Login e busca do estoque na API
            token = obter_token_autocerto(usuario, senha)
            estoque_api = buscar_estoque_autocerto(token, unidade_id)
            print(f" -> Veículos recebidos da AutoCerto: {len(estoque_api)}")
            
            api_map = {int(v["Codigo"]): v for v in estoque_api}
            
            # 2. Carrega estoque atual desta loja no Firestore para comparação
            docs_banco = db.collection("estoque").where("unidade_id", "==", loja_id).stream()
            banco_map = {}
            for doc in docs_banco:
                banco_map[int(doc.id)] = doc.to_dict()
                
            print(f" -> Veículos existentes no Firestore para esta loja: {len(banco_map)}")
            
            # 3. Classificação e Validação
            novos = []
            atualizados = []
            inalterados = []
            inativados = []
            
            batch = db.batch()
            batch_count = 0
            
            # Processa veículos vindos da API
            for cod_api, carro_api in api_map.items():
                doc_ref = db.collection("estoque").document(str(cod_api))
                data_mod_api = str(carro_api.get("DataModificacao") or "")
                
                if cod_api not in banco_map:
                    # NOVO
                    novos.append(cod_api)
                    doc_payload = montar_documento_veiculo(carro_api, loja_id, loja_nome, status="ATIVO", ativo=True, acao_sync="INSERIDO")
                    batch.set(doc_ref, doc_payload)
                    batch_count += 1
                else:
                    doc_antigo = banco_map[cod_api]
                    data_mod_antiga = str(doc_antigo.get("data_modificacao_api") or "")
                    estava_inativo = not doc_antigo.get("ativo", True)
                    
                    if data_mod_api > data_mod_antiga or estava_inativo:
                        atualizados.append(cod_api)
                        doc_payload = montar_documento_veiculo(carro_api, loja_id, loja_nome, status="ATIVO", ativo=True, acao_sync="ATUALIZADO")
                        batch.set(doc_ref, doc_payload, merge=True)
                        batch_count += 1
                    else:
                        inalterados.append(cod_api)
                
                if batch_count >= 400:
                    batch.commit()
                    batch = db.batch()
                    batch_count = 0

            # Processa veículos que saíram do estoque (Soft Delete)
            for cod_banco, doc_banco in banco_map.items():
                if cod_banco not in api_map and doc_banco.get("ativo", True):
                    inativados.append(cod_banco)
                    doc_ref = db.collection("estoque").document(str(cod_banco))
                    batch.update(doc_ref, {
                        "ativo": False,
                        "status": "INATIVO",
                        "ultima_acao_sync": "INATIVADO",
                        "updated_at": datetime.now(timezone.utc).isoformat()
                    })
                    batch_count += 1
                    
                    if batch_count >= 400:
                        batch.commit()
                        batch = db.batch()
                        batch_count = 0

            if batch_count > 0:
                batch.commit()

            print(f"\n Resultado da sincronização de {loja_nome}:")
            print(f"  • Novos inseridos:       {len(novos)}")
            print(f"  • Atualizados:           {len(atualizados)}")
            print(f"  • Inalterados:           {len(inalterados)}")
            print(f"  • Inativados (Vendidos): {len(inativados)}")
            
        except Exception as e:
            print(f" [X] Erro ao sincronizar a loja {loja_nome}: {e}")

    print("\n Sincronização multi-loja concluída com sucesso!")

if __name__ == "__main__":
    main()
