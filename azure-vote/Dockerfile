FROM tiangolo/uwsgi-nginx-flask:python3.9
RUN pip install redis azure.identity azure.keyvault.secrets
ADD /azure-vote /app