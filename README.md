# cads-build-farm

This repository manages the "build and push" GitHub Actions for all the services in CADS project. This GitOps part is an important component of the whole CI/CD pipeline that includes also FluxCD configuration on [cads-deployment](https://github.com/ecmwf-projects/cads-deployment).

## CI/CD Workflow
![CopDS CI_CD drawio](https://user-images.githubusercontent.com/59499702/220074303-4a89cc7d-76a3-4f73-8cf4-5fd748760b0b.png)

The current pipeline is designed in order to have the **Continuous Deployment** part fully automated. At the moment, the **Integration** part remains manual to grant the developers more control on releases. 

#### FluxCD automated Deployment
As described [here](link_to_cads-deployment_readme) FluxCD automatically reconcile what's present in a source with the status of a Kubernetes cluster. In this pipeline this tool has two scopes: with two dedicated configurations, Flux is able to keep updated each service inside the `cds2-dev` namespace and deploy on `cds2-test` only the tagged images. This setting allow us to have a continuously-deployed dev environment with the last changes on the code of all repositories and have a production environment (`cds2-test`) updated only with images that match with a specific tag (release versioning).

#### Integration tests
To verify the proper functioning of the code deployed on a specific namespace the user(developer) could run manually the integration tests from the [cads-api-client repo](link_to_cads-api-client-repo). The integration tests check ... of the services and api exposed.

#### Build and Push
Once that the developer is satisfied with all the changes the [Build all images](link_to_action) could be triggered manually. During this step is possibile to specify the tag to use for the build. This will automatically trigger the "build and push" workflow of each CADS service that build the image with the GitHub BuildX provided engine and push the builded image on the Harbor private registry with a `tag`. During the manual triggering step is possibile to specify the `tag` to use for the build. 

#### Tag release commit
Once that all the images are builded and stored on the private registry the next step is to change the Kustomize defnition files specifying the `tag` of the version for the images used into `cds2-test` namespace. This will trigger the deploy on that namespace with the new images selected.

### Release pipeline example
At the end of a Work Package cicle we want to release the version `2023.02.0` of the CADS portal.

1. All the developers merges their working branches on main of each service repo. This will trigger Flux.
2. FluxCD configuration updates `cds2-dev` namespace inside the Kubernets cluster with all the changes.
3. Run the integration tests as described from the cads-api-client repo README.md

        cd <YOUR_PATH>/cads-api-client
        conda activate DEVELOP

        CADS_API_URL=http://cds2-dev.copernicus-climate.eu/api make integration-tests

    If somehow anything is broken or some of the tests have <span style="color:red">**FAILED**</span> status you can safely check it, fix and commit changes on the related repo
4. Now that all the test have <span style="color:green">**PASSED**</span> status and everything works as expected, we can manually trigger the GitHub Action `Build all images` on this repo from the Action panel. I specify that the tag will be `2023-02.0`

    ACTION_PANEL_SCREENSHOT

    Wait until all the workflows are finished

    WORKFLOW_STATUS_SCREENSHOT

5. Wait until all the workflows are finished. Now from `cads-deployment` repo edit this file

        cads-deployment/kustomize/overlays/test/kustomization.yaml

    and change the `newTag` parameter for all the related services under the `images` list

        images:
            - name: cads-broker
            newName: my-private-registry/broker
            newTag: '2023-02.0'
            - name: catalogue-api
            newName: my-private-registry/catalogue-api-service
            newTag: '2023-02.0'
            - name: cads-catalogue-manager
            [...]

    Commit to trigger Flux Automated Deployment
6. FluxCD configuration updates `cds2-test` namespace inside the Kubernets cluster. All the components will be depoyed using the selected image. Now you can set the version tag also in Flux configuration file. 

    To do that edit this file

        cads-deployment/clusters/ecmwf/test/cads-deployment-test-source.yaml

    and change the value for the `ref` parameter to `2023-02.0`

