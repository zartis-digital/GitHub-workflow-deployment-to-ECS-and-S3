# ECS/S3 deployment of a React/NodeJS app via GitHub workflows

> “I'd far rather be happy than right any day.”

The file `.github/workflows/production.yml` contains the workflow definition for CD of a NodeJS express app to ECS and its React client to an S3 bucket.

Client and server live on separate directories. In this case: `server` for the server, and `client` for the client.

## Flow trigger

Any push onto `master` of the GitHub repo will trigger the flow:

```
on:
  push:
    branches:
      - master
```

## Test and create production build of client

To establish a working directory for the job, we need to set and `env` variable:

```
    # Establishes the working directory
    env:
      working-directory: client
```

To Install dependencies we call `yarn install` in the working directory:

```
      - name: Client Yarn Install
        working-directory: ${{ env.working-directory }}
        run: |
          yarn install
```

Then we run the unit/integration tests. We need to set `CI=true` so that Jest bails after running the tests:

```
      - name: Client Unit Tests
        working-directory: ${{ env.working-directory }}
        run: |
          CI=true yarn test
```

We need to create a `build` directory so that the workflow is aware of its existence. The directory created in the build step is not available to other steps in the workflow:

```
      - name: Make build directory
        run: mkdir ./client/build
```

We build the client:

```
      - name: Client build
        working-directory: ${{ env.working-directory }}
        run: |
          yarn build
```

And lastly we upload the build as an artifact and name it `clientbuild`:

```
      - name: Upload build artifact
        uses: actions/upload-artifact@v1
        with:
          name: clientbuild
          path: client/build
```

## Install, test and deploy server

Just like above, we need to:
* Establish a working directory
* Install the dependencies for the server
* Run the tests on the server 

```
    env:
      working-directory: server
    steps:
      - uses: actions/checkout@v1
      - name: Server Yarn Install
        working-directory: ${{ env.working-directory }}
        run: |
          yarn install
      - name: Server Unit Tests
        working-directory: ${{ env.working-directory }}
        run: |
          CI=true yarn test
```

### Configure AWS credentials and login to AWS ECR repo

These steps will configure AWS credentials and login to the AWS ECR repo. `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_REGION` are stored as secrets in the secrets section of the GitHub repo.

```
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
```

### Create Docker image and push to repo

We're tagging the image with the GitHub `run_id`. We could also be tagging with `latest`, or any other tag we wish. The previous step, with `id = login-ecr`, outputs the registry name, which is contained in `steps.login-ecr.outputs.registry`

This step is labeled with `build_image` and will output a parameter called `image` with the current tag.

```
      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        working-directory: ${{ env.working-directory }}
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
          IMAGE_TAG: ${{ github.run_id }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
```

### Deploy new Task Definition to ECS

The first steps takes a template task definition (`ecs-task-definition.yson`) and outputs a JSON with a container named `container-name` and fills in the image field with the output of the previous `build-image` step.

The second one updates the `AWS_ECS_SERVICE` on the ECS `AWS_ECS_CLUSTER` cluster.

```
      - name: Render Amazon ECS task definition
        id: render-web-container
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: 
          ecs-task-definition.json
          container-name: <container-name>
          image: ${{ steps.build-image.outputs.image }}
      - name: Deploy to Amazon ECS service
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.render-web-container.outputs.task-definition }}
          service: ${{ secrets.AWS_ECS_SERVICE }}
          cluster: ${{ secrets.AWS_ECS_CLUSTER }}
```

The new revision of the Task Definition will update the service and trigger a re-deploy.

You can find more info on [Github actions and AWS Fargate](https://aws.amazon.com/blogs/opensource/github-actions-aws-fargate/) and [ECS: Updating a Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/update-service.html) in the docs, but good luck using it!

## Deploy client to S3 bucket

This job will only run if the previous two jobs are successful, hence the `needs` attribute. First it will download the `clientbuild` artifact. The results will be uploaded to the `AWS_PRODUCTION_BUCKET_NAME` bucket, given the correct credentials.

```
    name: Deploy client
    needs: [job_1, job_2]
    runs-on: ubuntu-latest
    steps:
      - name: Download result for job 1
        uses: actions/download-artifact@v1
        with:
          name: clientbuild
      - name: Deploy to S3
        uses: jakejarvis/s3-sync-action@master
        with:
          args: --acl public-read --delete
        env:
          AWS_S3_BUCKET: ${{ secrets.AWS_PRODUCTION_BUCKET_NAME }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: ${{ secrets.AWS_REGION }}
          SOURCE_DIR: "clientbuild"
```

## TO-DOs

The server Docker image shouldn't be pushed to ERC if the client build fails.
