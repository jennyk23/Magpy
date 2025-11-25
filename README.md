# Sistema de Controle Estufa Inteligente
# SISTEMA DE CONTROLE DE ESTUFA INTELIGENTE 
# Controle de temperatura, umidade e irrigação com registro em banco de dados
# ============================================================================
# Melhorias: Logging, threading, configurações dinâmicas, exportação JSON, simulação aprimorada.

import sqlite3
import json
import random
import time
import threading
import logging
from datetime import datetime

# ============================================================================
# CONFIGURAÇÃO DE LOGGING
# ============================================================================
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

# ============================================================================
# 1. CLASSE PARA GERENCIAR BANCO DE DADOS
# ============================================================================
class BancoEstufa:
    """Gerencia o banco de dados SQLite para o sistema de estufa"""
    
    def __init__(self, nome_db="estufa_dados.db"):
        self.nome_db = nome_db
        self._criar_tabelas()
    
    def _conexao(self):
        return sqlite3.connect(self.nome_db, detect_types=sqlite3.PARSE_DECLTYPES | sqlite3.PARSE_COLNAMES)
    
    def _criar_tabelas(self):
        """Cria as tabelas do banco se não existirem"""
        with self._conexao() as conn:
            conn.execute('''
                CREATE TABLE IF NOT EXISTS medicoes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                    temperatura REAL NOT NULL,
                    umidade_ar REAL NOT NULL,
                    umidade_solo REAL NOT NULL,
                    bomba_ativa INTEGER NOT NULL
                )
            ''')
            conn.execute('''
                CREATE TABLE IF NOT EXISTS alertas (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                    tipo_alerta TEXT NOT NULL,
                    mensagem TEXT NOT NULL,
                    valor_medido REAL
                )
            ''')
            conn.execute('''
                CREATE TABLE IF NOT EXISTS configuracoes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    parametro TEXT UNIQUE NOT NULL,
                    valor TEXT NOT NULL
                )
            ''')
    
    def inserir_medicao(self, temperatura, umidade_ar, umidade_solo, bomba_ativa):
        """Insere uma nova medição no banco"""
        with self._conexao() as conn:
            conn.execute('''
                INSERT INTO medicoes (temperatura, umidade_ar, umidade_solo, bomba_ativa)
                VALUES (?, ?, ?, ?)
            ''', (temperatura, umidade_ar, umidade_solo, int(bool(bomba_ativa))))
    
    def inserir_alerta(self, tipo_alerta, mensagem, valor_medido=None):
        """Insere um novo alerta no banco"""
        with self._conexao() as conn:
            conn.execute('''
                INSERT INTO alertas (tipo_alerta, mensagem, valor_medido)
                VALUES (?, ?, ?)
            ''', (tipo_alerta, mensagem, valor_medido))
    
    def obter_configuracao(self, parametro, default=None):
        """Obtém o valor de uma configuração"""
        with self._conexao() as conn:
            cursor = conn.execute('SELECT valor FROM configuracoes WHERE parametro = ?', (parametro,))
            row = cursor.fetchone()
            return row[0] if row else default
    
    def definir_configuracao(self, parametro, valor):
        """Define ou atualiza uma configuração"""
        with self._conexao() as conn:
            conn.execute('''
                INSERT INTO configuracoes (parametro, valor)
                VALUES (?, ?)
                ON CONFLICT(parametro) DO UPDATE SET valor=excluded.valor
            ''', (parametro, str(valor)))
    
    def exportar_medicoes_json(self, arquivo="medicoes_export.json", limite=None):
        """Exporta medições para JSON"""
        with self._conexao() as conn:
            query = 'SELECT id, timestamp, temperatura, umidade_ar, umidade_solo, bomba_ativa FROM medicoes ORDER BY id DESC'
            if limite:
                query += ' LIMIT ?'
                cursor = conn.execute(query, (limite,))
            else:
                cursor = conn.execute(query)
            linhas = cursor.fetchall()
        
        registros = []
        for r in linhas[::-1]:  # Ordem cronológica
            registros.append({
                "id": r[0],
                "timestamp": r[1],
                "temperatura": r[2],
                "umidade_ar": r[3],
                "umidade_solo": r[4],
                "bomba_ativa": bool(r[5])
            })
        with open(arquivo, "w", encoding="utf-8") as f:
            json.dump(registros, f, ensure_ascii=False, indent=2, default=str)
        logging.info(f"Exportadas {len(registros)} medições para {arquivo}")
        return arquivo

# ============================================================================
# 2. CLASSE PARA CONTROLE DA ESTUFA (INTEGRANDO SIMULAÇÃO E LÓGICA)
# ============================================================================
class ControleEstufa:
    """Controla a lógica da estufa inteligente, com simulação integrada"""
    
    def __init__(self, banco):
        self.banco = banco
        self.bomba_ativa = False
        self._parar = threading.Event()
        self._thread = None
        
        # Configurações padrão (carregadas se não existirem)
        self._set_config_default("intervalo_segundos", "5")
        self._set_config_default("temp_min", "18.0")
        self._set_config_default("temp_max", "30.0")
        self._set_config_default("umidade_solo_min", "30.0")
        self._set_config_default("umidade_ar_min", "30.0")
        self._set_config_default("regra_auto_bomba", "True")
        self._set_config_default("sim_variacao_temp", "0.8")
        self._set_config_default("sim_variacao_umidade", "1.5")
    
    def _set_config_default(self, parametro, valor):
        if self.banco.obter_configuracao(parametro) is None:
            self.banco.definir_configuracao(parametro, valor)
    
    def obter_config(self, parametro, tipo=str):
        v = self.banco.obter_configuracao(parametro)
        if v is None:
            return None
        if tipo == float:
            return float(v) if v.replace('.', '').isdigit() else None
        if tipo == int:
            return int(v) if v.isdigit() else None
        if tipo == bool:
            return v.lower() in ("1", "true", "yes", "sim", "y")
        return v
    
    def definir_config(self, parametro, valor):
        self.banco.definir_configuracao(parametro, valor)
        logging.info(f"Config '{parametro}' setada para {valor}")
    
    def ler_sensores_simulados(self):
        """Simula leituras de sensores com variações configuráveis"""
        base_temp = 24.0
        base_umid_ar = 50.0
        base_umid_solo = 45.0
        var_t = self.obter_config("sim_variacao_temp", float) or 0.8
        var_u = self.obter_config("sim_variacao_umidade", float) or 1.5
        
        temperatura = round(base_temp + random.uniform(-var_t, var_t), 2)
        umidade_ar = round(base_umid_ar + random.uniform(-var_u, var_u), 2)
        umidade_solo = round(base_umid_solo + random.uniform(-var_u * 2, var_u * 2), 2)
        return temperatura, umidade_ar, umidade_solo
    
    def verificar_condicoes(self):
        """Verifica condições, toma decisões e registra medição"""
        temp, umid_ar, umid_solo = self.ler_sensores_simulados()
        
        # Regras configuráveis
        temp_min = self.obter_config("temp_min", float) or 18.0
        temp_max = self.obter_config("temp_max", float) or 30.0
        umid_solo_min = self.obter_config("umidade_solo_min", float) or 30.0
        umid_ar_min = self.obter_config("umidade_ar_min", float) or 30.0
        auto_bomba = self.obter_config("regra_auto_bomba", bool)
        
        # Alertas de temperatura
        if temp > temp_max:
            self.banco.inserir_alerta("Temperatura", f"Temperatura alta: {temp}°C > {temp_max}°C", temp)
        elif temp < temp_min:
            self.banco.inserir_alerta("Temperatura", f"Temperatura baixa: {temp}°C < {temp_min}°C", temp)
        
        # Lógica de irrigação
        if umid_solo < umid_solo_min:
            self.banco.inserir_alerta("Irrigação", f"Umidade do solo baixa: {umid_solo}% < {umid_solo_min}%", umid_solo)
            if auto_bomba:
                self.ligar_bomba()
        else:
            self.desligar_bomba()
        
        # Alerta de umidade do ar
        if umid_ar < umid_ar_min:
            self.banco.inserir_alerta("Umidade Ar", f"Umidade do ar baixa: {umid_ar}% < {umid_ar_min}%", umid_ar)
        
        # Registrar medição
        self.banco.inserir_medicao(temp, umid_ar, umid_solo, self.bomba_ativa)
        logging.info(f"Medição: T={temp}°C, UA={umid_ar}%, US={umid_solo}%, Bomba={'Ativa' if self.bomba_ativa else 'Inativa'}")
        return temp, umid_ar, umid_solo, self.bomba_ativa
    
    def ligar_bomba(self):
        if not self.bomba_ativa:
            self.bomba_ativa = True
            logging.info("Bomba ligada (simulada)")
    
    def desligar_bomba(self):
        if self.bomba_ativa:
            self.bomba_ativa = False
            logging.info("Bomba desligada (simulada)")
    
    def iniciar_loop(self):
        if self._thread and self._thread.is_alive():
            logging.warning("Loop já em execução")
            return
        self._parar.clear()
        self._thread = threading.Thread(target=self._loop_continuo, daemon=True)
        self._thread.start()
        logging.info("Loop de aquisição iniciado")
    
    def _loop_continuo(self):
        intervalo = self.obter_config("intervalo_segundos", float) or 5.0
        while not self._parar.is_set():
            start = time.time()
            try:
                self.verificar_condicoes()
            except Exception as e:
                logging.exception(f"Erro no ciclo: {e}")
                self.banco.inserir_alerta("Erro", str(e))
            elapsed = time.time() - start
            self._parar.wait(max(0, intervalo - elapsed))
        logging.info("Loop finalizado")
    
    def parar_loop(self):
        if self._thread and self._thread.is_alive():
            self._parar.set()
            self._thread.join(timeout=5.0)
            logging.info("Loop parado")

# ============================================================================
# 3. FUNÇÃO PRINCIPAL
# ============================================================================
def main():
    """Função principal do sistema"""
    banco = BancoEstufa()
    controle = ControleEstufa(banco)
    
    print("Sistema de Controle de Estufa Inteligente Iniciado (Otimizado)")
    print("Pressione Ctrl+C para parar")
    
    try:
        controle.iniciar_loop()
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\nSistema parado pelo usuário")
    finally:
        controle.desligar_bomba()
        controle.parar_loop()
        # Exemplo: Exportar últimas 200 medições
        try:
            arquivo = banco.exportar_medicoes_json(limite=200)
            print(f"Exportado: {arquivo}")
        except Exception as e:
            logging.exception(f"Falha na exportação: {e}")

if __name__ == "__main__":
    main()
