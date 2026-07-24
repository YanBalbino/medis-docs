# Contexto do Sistema

## Propósito

O Medis é um sistema digital de pré-triagem e encaminhamento de pacientes que utiliza regras estruturais e modelos de Aprendizagem de Máquina treinados para apoiar esses fluxos na área da saúde.

## Problema que sistema resolve

Sistema centraliza:

- autenticação de usuários
- consulta de doenças e informações associadas
- triagem baseada em questionários
- triagem baseada em modelos de predição
- registro de histórico de triagens
- administração de dados clínicos e operacionais

## Atores principais

- **Paciente/usuário final** — realiza cadastro, login, consulta informações e executa triagens
- **Administrador** — mantém doenças, perguntas, sintomas, critérios, unidades e modelos
- **Sistema de backend** — orquestra regras de negócio, persistência e integrações
- **Serviço de modelos** — expõe descrição e execução de modelos preditivos

## Princípios para documentação

- visão macro primeiro
- detalhes específicos depois
- rastreabilidade para código real
- linguagem orientada a domínio
- atualização incremental

## Escopo desta seção

Esta seção descreve sistema em nível conceitual. Detalhes de implementação ficam nas páginas de cada repositório.
