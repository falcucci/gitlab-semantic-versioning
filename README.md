# gitlab-semantic-versioning

Docker image that can be used to automatically version projects using semantic versioning. 

Visit [semver.org](https://semver.org/) to read more about semantic versioning.

## How is the version determined?

Versions are being maintained using git tags.

If no git tag is available, the first version update will result in version 1.0.0.
If git tags are available, it will determine whether to do a major, minor, or patch update based on specific merge request labels. The `bump-minor` and `bump-major` labels exist to do either a minor or major bump. If a merge request has no labels attached, it will perform a patch update by default.

## Prerequisites

### Group labels

As stated above, the version update workflow relies on merge request labels to determine the new version. The `bump-minor` and `bump-major` labels have been set as global GitLab labels. However, global labels only propogate to groups created after setting a global label. When adding a global label, they [do not automatically propogate to existing groups](https://gitlab.com/gitlab-org/gitlab-ce/issues/12707).

If you cannot select the specified labels in your merge request, your group was most likely created before the global labels were defined. Please follow [this guide to setup group-specific labels](https://docs.gitlab.com/ee/user/project/labels.html) 

### API token and group

To extract the labels from merge requests, we need an API token to access the Gitlab API. Unfortunately, [GitLab doesn't yet support non-user specific access tokens](https://gitlab.com/gitlab-org/gitlab-ee/issues/756). 

Ask your GitLab administrator to add a dummy user `${group_name}_npa` to GitLab with access only to your project group. Log in with this user, and create a [personal access token](https://gitlab.wbaa.pl.ing.net/profile/personal_access_tokens) with api scope access.

Copy the generated API token and keep it available for the next section.

### Group-level variables

The NPA username and token need to be injected into the version-update container as environment variables. For this, we'll use group-level variables. 

Go to your group's variables section under `Settings` -> `CI / CD`.

Add the following variables:

| Key             | Value                                                                |
|-----------------|----------------------------------------------------------------------|
| NPA_USERNAME    | The name of the NPA user created for your group: `${group_name}_npa` |
| NPA_PASSWORD    | The personal access token with API scope generated for the NPA user  |
| NPA_EMAIL       | git NPA user email  |
| NPA_NAME        | git NPA user name  |



## Pipeline configuration

The pipeline configuration below will:
1. Generate a unique version tag based on git describe
2. Update the version for every build on the `master` branch.

This pipeline omits steps for building the  project and pushing the resulting Docker image to the registry.

```
stages:
  - generate-env-vars
  - version
  - tag-latest

variables:
  REPOSITORY_URL: https://$NPA_USERNAME:$NPA_PASSWORD@gitlab.luizalabs.com/$CI_PROJECT_PATH.git 

version:
  stage: version
  image: falcucci/gpython:1.0.0
  script:
    - python3 /version-update/version-update.py
  only:
    refs:
      - master
    variables:
      - $CI_COMMIT_MESSAGE =~ /See merge request/
      
# for node.js applications
package:
  stage: package
  before_script:
    - git reset --hard
    - git checkout $CI_COMMIT_REF_NAME
    - git pull origin $CI_COMMIT_REF_NAME
    - git config --global user.email $NPA_EMAIL
    - git config --global user.name $NPA_NAME
    - git remote set-url --push origin $REPOSITORY_URL
  script:
    - NTAG=`eval 'git describe --tags $(git rev-list --tags --max-count=1)'`
    - npm version --no-git-tag-version $NTAG
    - git commit -a -m "$NTAG"
    - git push origin HEAD:$CI_COMMIT_REF_NAME
  only:
    refs:
      - master
    variables:
      - $CI_COMMIT_MESSAGE =~ /See merge request/
```

## Merge request instructions

### Squash commits when merge request is accepted

The new version will be determined based on the commit message. GitLab will automatically format a merge request commit message if the 'Squash commits when merge request is accepted` checkbox is checked during merge request creation.

This workflow relies on that commit message format and will fail the pipeline if it cannot extract the merge request id from the commit message. 

Unfortunately, GitLab [doesn't yet allow for setting the checkbox to checked by default](https://gitlab.com/gitlab-org/gitlab-ce/issues/27956). Until implemented, make sure to manually check the squash option upon creation.

### Add a label to indicate a minor or major update

As described above, if you want to perform a minor or major update, don't forget to add the appropriate label to your merge request.
