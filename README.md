# hooke
## F = -kx

Docker (pull image, start/stop) de forma automatizada, através de um pipeline CI/CD, unindo Github Actions, Watch Tower e Send Grid. 

Cenário: Desenvolvedores - ou você mesmo - atualizam imagens docker para um registry, e você - ou o próprio dev - com um acesso SSH ao Docker host, onde estão os containers, agendam aquele tempinho para fazer a implantação da nova atualização com um docker pull image, seguido de docker stop container , docker rm container e docker start container

1. Build da imagem Docker (CI);
2. Push dessa imagem para um registry (CD - Continuous Delivery);
3. Notificação, por e-mail, da entrada de uma nova imagem no registry (passo anterior);
4. Pull da imagem e atualização dos containers (CD - Continuous Deployment);
5. Notificação, por e-mail, da atualização dos containers (passo anterior);

Com estes passos, todas as vezes que uma atualização de imagens no repositório ocorrer, ela irá ser testada e enviada para o registry e, automaticamente, será feita a atualização (pull) dessa imagem no Docker host que a hospeda, além disso, os containers também serão atualizados.

Devemos criar o seguinte workflow, dentro da pasta .github/workflows:

### Pipeline (Integration/Delivery)

```yml
name: Workflow - sendgrid pull push

on:
  push:
    branches:
      - master
  pull_request:

env:
  # Variavel de ambiente vista para qualquer job
  # nome da imagem - altere para o nome correto
  IMAGE_NAME: nome-imagem

jobs:
  # Teste de build
  # Ver mais https://docs.docker.com/docker-hub/builds/automated-testing/
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Executando testes
       # Testa se o arquivo 'builda'
        run: |
          if [ -f docker-compose.test.yml ]; then
            docker-compose --file docker-compose.test.yml build
            docker-compose --file docker-compose.test.yml run sut
          else
            docker build . --file Dockerfile
          fi
  # Push da imagem para o registry do github
  push:
    # Condicao para somente fazer o push se o job test executou com sucesso
    needs: test

    runs-on: ubuntu-18.04
    # Só executa se o evento atual for push (e não um PR)
    if: github.event_name == 'push'

    steps:
      - uses: actions/checkout@v2

      - name: Build image
        # chamada da variavel IMAGE_NAME definida em env 
        run: docker build . --file Dockerfile --tag $IMAGE_NAME

      - name: Login no GitHub Container Registry
      # Personal Access Token (PAT) criado e adicionado no Actions Secrets.
        run: echo "${{ secrets.CR_PAT }}" | docker login https://ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Push da imagem para GitHub Container Registry
        run: |
          IMAGE_ID=ghcr.io/${{ github.repository_owner }}/$IMAGE_NAME
          # Transformar nome da imagem de maiúsculas para minúsculas 
          IMAGE_ID=$(echo $IMAGE_ID | tr '[A-Z]' '[a-z]')
          # Retira o prefixo git ref da versão
          VERSION=$(echo "${{ github.ref }}" | sed -e 's,.*/\(.*\),\1,')
          # Retirar prefixo v da tag
          [[ "${{ github.ref }}" == "refs/tags/"* ]] && VERSION=$(echo $VERSION | sed -e 's/^v//')
          # Use tag latest como convenção para docker:latest
          [ "$VERSION" == "master" ] && VERSION=latest
          echo IMAGE_ID=$IMAGE_ID
          echo VERSION=$VERSION
          # comandos finais para push
          docker tag $IMAGE_NAME $IMAGE_ID:$VERSION
          docker push $IMAGE_ID:$VERSION
  
  email:
    # Condicao para somente fazer o push se o anterior executou com sucesso
    needs: push
    runs-on: ubuntu-18.04

    steps:
      - name: SendGrid Action
        uses: mmichailidis/sendgrid-mail-action@v1.0
        with:
          sendgrid-token: ${{ secrets.SENDGRID_API_KEY }}
          mail: mail@example.com,outroemail@example.com #email de destino
          #from definido no sendgrid - https://sendgrid.com/docs/API_Reference/SMTP_API/integrating_with_the_smtp_api.html
          from: ${{ secrets.SENDGRID_MAILFROM }}
          subject: Nova imagem no registry
          text: Uma nova imagem foi enviada para o registry. Aguarde a notificação, do serviço Watch Tower, da implantação automática na produção!
```

Esse workflow contempla as ações 1, 2 e 3, que correspondem ao build (CI), push da imagem no registry (CD) e notificação via e-mail.
