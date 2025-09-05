import hashlib
import json
from time import time
from urllib.parse import urlparse
from uuid import uuid4

import requests
from flask import Flask, jsonify, request

class Blockchain:
    def __init__(self):
        """
        Construtor da classe Blockchain.
        """
        self.current_transactions = []
        self.chain = []
        self.nodes = set()

        # Cria o bloco Gênese (o primeiro bloco da cadeia)
        self.new_block(previous_hash='1', proof=100)

    def register_node(self, address: str):
        """
        Adiciona um novo nó à lista de nós da rede.
        :param address: Endereço do nó. Ex: 'http://192.168.0.5:5000'
        """
        parsed_url = urlparse(address)
        if parsed_url.netloc:
            self.nodes.add(parsed_url.netloc)
        elif parsed_url.path:
            # Aceita um URL como '192.168.0.5:5000'.
            self.nodes.add(parsed_url.path)
        else:
            raise ValueError('URL inválido')

    def valid_chain(self, chain: list) -> bool:
        """
        Determina se uma dada blockchain é válida.
        :param chain: Uma blockchain.
        :return: True se for válida, False se não.
        """
        last_block = chain[0]
        current_index = 1

        while current_index < len(chain):
            block = chain[current_index]
            print(f'{last_block}')
            print(f'{block}')
            print("\n-----------\n")

            # Verifica se o hash do bloco está correto
            if block['previous_hash'] != self.hash(last_block):
                return False

            # Verifica se a Prova de Trabalho está correta
            if not self.valid_proof(last_block['proof'], block['proof'], self.hash(last_block)):
                return False

            last_block = block
            current_index += 1

        return True

    def resolve_conflicts(self) -> bool:
        """
        Este é o nosso algoritmo de consenso. Ele resolve conflitos
        substituindo nossa cadeia pela mais longa da rede.
        :return: True se nossa cadeia foi substituída, False se não.
        """
        neighbours = self.nodes
        new_chain = None

        # Estamos apenas procurando por cadeias mais longas que a nossa
        max_length = len(self.chain)

        # Pega e verifica as cadeias de todos os nós da nossa rede
        for node in neighbours:
            try:
                response = requests.get(f'http://{node}/chain')

                if response.status_code == 200:
                    length = response.json()['length']
                    chain = response.json()['chain']

                    # Verifica se o comprimento é maior e se a cadeia é válida
                    if length > max_length and self.valid_chain(chain):
                        max_length = length
                        new_chain = chain
            except requests.exceptions.ConnectionError:
                print(f"Não foi possível conectar ao nó {node}.")


        # Substitui nossa cadeia se descobrirmos uma nova cadeia válida e mais longa
        if new_chain:
            self.chain = new_chain
            return True

        return False

    def new_block(self, proof: int, previous_hash: str = None) -> dict:
        """
        Cria um novo Bloco na Blockchain.
        :param proof: A prova dada pelo algoritmo de Prova de Trabalho.
        :param previous_hash: Hash do Bloco anterior.
        :return: Novo Bloco.
        """
        block = {
            'index': len(self.chain) + 1,
            'timestamp': time(),
            'transactions': self.current_transactions,
            'proof': proof,
            'previous_hash': previous_hash or self.hash(self.chain[-1]),
        }

        # Reseta a lista de transações atuais
        self.current_transactions = []

        self.chain.append(block)
        return block

    def new_transaction(self, sender: str, recipient: str, amount: float) -> int:
        """
        Cria uma nova transação para ir para o próximo Bloco minerado.
        :param sender: Endereço do remetente.
        :param recipient: Endereço do destinatário.
        :param amount: Quantidade.
        :return: O índice do Bloco que conterá esta transação.
        """
        self.current_transactions.append({
            'sender': sender,
            'recipient': recipient,
            'amount': amount,
        })

        return self.last_block['index'] + 1

    @property
    def last_block(self) -> dict:
        """
        Retorna o último Bloco da cadeia.
        """
        return self.chain[-1]

    @staticmethod
    def hash(block: dict) -> str:
        """
        Cria um hash SHA-256 de um Bloco.
        :param block: Bloco.
        :return: String com o hash.
        """
        # Devemos garantir que o Dicionário esteja Ordenado, ou teremos hashes inconsistentes
        block_string = json.dumps(block, sort_keys=True).encode()
        return hashlib.sha256(block_string).hexdigest()

    def proof_of_work(self, last_block: dict) -> int:
        """
        Algoritmo de Prova de Trabalho simples:
         - Encontre um número 'p' tal que o hash(hash_anterior + p) contenha 4 zeros à esquerda.
        :param last_block: O último bloco da cadeia.
        :return: O número 'p' (prova).
        """
        last_proof = last_block['proof']
        last_hash = self.hash(last_block)

        proof = 0
        while self.valid_proof(last_proof, proof, last_hash) is False:
            proof += 1

        return proof

    @staticmethod
    def valid_proof(last_proof: int, proof: int, last_hash: str) -> bool:
        """
        Valida a prova: O hash(last_proof, proof, last_hash) contém 4 zeros à esquerda?
        :param last_proof: Prova anterior.
        :param proof: Prova atual.
        :param last_hash: Hash do bloco anterior.
        :return: True se correto, False se não.
        """
        guess = f'{last_proof}{proof}{last_hash}'.encode()
        guess_hash = hashlib.sha256(guess).hexdigest()
        # A dificuldade é definida pelo número de zeros. Pode ser ajustada.
        return guess_hash[:4] == "0000"


# --- Início da API com Flask ---

# Instancia nosso Nó
app = Flask(__name__)

# Gera um endereço globalmente único para este nó
node_identifier = str(uuid4()).replace('-', '')

# Instancia a Blockchain
blockchain = Blockchain()


@app.route('/mine', methods=['GET'])
def mine():
    # Executamos o algoritmo de prova de trabalho para obter a próxima prova...
    last_block = blockchain.last_block
    proof = blockchain.proof_of_work(last_block)

    # Devemos receber uma recompensa por encontrar a prova.
    # O remetente é "0" para significar que este nó minerou uma nova moeda.
    blockchain.new_transaction(
        sender="0",
        recipient=node_identifier,
        amount=1,
    )

    # Forja o novo Bloco adicionando-o à cadeia
    previous_hash = blockchain.hash(last_block)
    block = blockchain.new_block(proof, previous_hash)

    response = {
        'message': "Novo Bloco Forjado",
        'index': block['index'],
        'transactions': block['transactions'],
        'proof': block['proof'],
        'previous_hash': block['previous_hash'],
    }
    return jsonify(response), 200


@app.route('/transactions/new', methods=['POST'])
def new_transaction():
    values = request.get_json()

    # Verifica se os campos necessários estão nos dados postados
    required = ['sender', 'recipient', 'amount']
    if not values or not all(k in values for k in required):
        return 'Valores ausentes', 400

    # Cria uma nova transação
    index = blockchain.new_transaction(values['sender'], values['recipient'], values['amount'])

    response = {'message': f'A transação será adicionada ao Bloco {index}'}
    return jsonify(response), 201


@app.route('/chain', methods=['GET'])
def full_chain():
    response = {
        'chain': blockchain.chain,
        'length': len(blockchain.chain),
    }
    return jsonify(response), 200


@app.route('/nodes/register', methods=['POST'])
def register_nodes():
    values = request.get_json()

    nodes = values.get('nodes')
    if nodes is None:
        return "Erro: Forneça uma lista válida de nós", 400

    for node in nodes:
        blockchain.register_node(node)

    response = {
        'message': 'Novos nós foram adicionados',
        'total_nodes': list(blockchain.nodes),
    }
    return jsonify(response), 201


@app.route('/nodes/resolve', methods=['GET'])
def consensus():
    replaced = blockchain.resolve_conflicts()

    if replaced:
        response = {
            'message': 'Nossa cadeia foi substituída',
            'new_chain': blockchain.chain
        }
    else:
        response = {
            'message': 'Nossa cadeia é autoritativa',
            'chain': blockchain.chain
        }

    return jsonify(response), 200


if __name__ == '__main__':
    from argparse import ArgumentParser

    parser = ArgumentParser()
    parser.add_argument('-p', '--port', default=5000, type=int, help='Porta para escutar')
    args = parser.parse_args()
    port = args.port

    app.run(host='0.0.0.0', port=port)
