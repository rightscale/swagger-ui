## Using swagger-ui for public API docs served from S3

For public facing swagger-ui API docs, we cannot serve from the `idocs.rightscale.com` location, used for internal docs. The files in these buckets lack the appropriate public ACLs for allowing public READ access.

Instead, we serve from the Tools account's `rs-api-docs` S3 bucket:
https://us-3.rightscale.com/acct/80694/clouds/1/s3_browser?ui_route=buckets/rs-api-docs/browser

Files in this location are publicly accessible with READ access, and are ultimately referenced by our external facing `https://reference.rightscale.com` URL.

_NOTE: The content served from `https://reference.rightscale.com` appears to be cached for 24 hours through CloudFront. Changes uploaded to the S3 bucket won't be seen through `https://reference.rightscale.com` for 24 hours, unless you bust the cache through an API. (Information on how to bust the cache TBD.)_

For example:
https://reference.rightscale.com/cost_management/#/index.html
serves content from app specific folders, such as below for `cost_management`, in the `rs-api-docs` S3 bucket:
```
rs-api-docs/cost_management
rs-api-docs/cost_management/index.html
rs-api-docs/cost_management/swagger.json
rs-api-docs/cost_management/swagger.yaml
```
where the `index.html`, uploaded from this swagger-ui repo's `/dist/index.html`, uses relative references `../swagger-ui/` to common/shared swagger-ui files in a parallel `swagger-ui` folder:
```
rs-api-docs/swagger-ui
rs-api-docs/swagger-ui/favicon-16x16.png
rs-api-docs/swagger-ui/favicon-32x32.png
rs-api-docs/swagger-ui/swagger-ui-bundle.js
rs-api-docs/swagger-ui/swagger-ui-bundle.js.map
rs-api-docs/swagger-ui/swagger-ui-standalone-preset.js
rs-api-docs/swagger-ui/swagger-ui-standalone-preset.js.map
rs-api-docs/swagger-ui/swagger-ui.css
rs-api-docs/swagger-ui/swagger-ui.css.map
rs-api-docs/swagger-ui/swagger-ui.js
rs-api-docs/swagger-ui/swagger-ui.js.map
```

#### How to set up your app's folder in S3

Let's suppose we're creating `my-awesome-new-app` and we want to reference it from:
`https://reference.rightscale.com/my-awesome-new-app/#/index.html`

Follow these steps:

1. Make your app specific folder directly under `/rs-api-docs`, parallel to the `/rs-api-docs/swagger-ui` folder.
```
rs-api-docs/my-awesome-new-app/		<--- new parallel folder for "my-awesome-new-app"
rs-api-docs/swagger-ui/           <--- common/shared swagger-ui files
```
_Note: The folder name must match the URL app reference in order for the generic `index.html` to dynamically figure out where/how to locate the app-specific `swagger.json/.yaml`. So, if your desired URL is:
`https://reference.rightscale.com/my-awesome-new-app/#/index.html`
then your folder name must be `my-awesome-new-app`.
In your URL reference to your app's `index.html`, you MUST also make use of the "/#/" syntax in order for proper parsing to occur. See [the comments here in github](https://github.com/rightscale/swagger-ui/blob/cc238a123604d1ffc158907822df21fea851dd66/dist/index.html#L98-L130) as to why._

2. Clone `git@github.com:rightscale/swagger-ui.git` to get its latest `dist/index.html`, and upload this `index.html` directly into your app-specific folder.
```
rs-api-docs/my-awesome-new-app/index.html
```
The file should require no modifications, it just needs to reside in your app-specific folder so that the location of the corresponding `swagger.json/.yaml` files can be dynamically parsed/inferred from the URL.
 
_Note: The other common/shared swagger-ui files in `/dist` are already uploaded and exist in the `/rs-api-docs/swagger-ui` S3 bucket/folder. You do not need to upload these._

3. Be sure that the `deploy:` section of your app's [`.travis.yaml`](https://github.com/rightscale/optima_analytics/blob/790775c5e46dca96eb82500479c4a3979ec797b3/.travis.yml#L90-L103) uploads the appropriate `swagger.json/.yaml` files to your app-specific folder in `rs-api-docs`:
```
rs-api-docs/my-awesome-new-app/swagger.json
rs-api-docs/my-awesome-new-app/swagger.yaml
```
_Note: Be sure you've defined the `S3_KEY_ID_RS_API_DOCS/S3_KEY_SECRET_RS_API_DOCS` id/key inputs for `rs-api-docs` in the Tools account in your Travis repo settings, like here: https://travis-ci.com/rightscale/optima_analytics/settings, as these differ from `idocs.rightscale.com` for internal docs which uses STAGE account id/key._
```
# Uploads static assets for public facing API docs
# only does this for 'docs' and 'production' branches.
- provider: s3
  bucket: rs-api-docs
  region: us-west-1
  skip_cleanup: true
  local_dir: $TRAVIS_BUILD_DIR/$WORK_DIR/generated/swagger/cost_management/swagger
  upload_dir: cost_management
  acl: public_read
  access_key_id: "$S3_KEY_ID_RS_API_DOCS"
  secret_access_key: "$S3_KEY_SECRET_RS_API_DOCS"
  on:
    branch: production
    condition: $WORK_DIR = front
```
_Note: This example above only pushes to `rs-api-docs` for the production branch, and relies on dockerhub webhooks to trigger this deploy for `production` travis "builds". To do so, make sure your app has `staging` and `production` branches in github (create manually from fast-forward of `master`, if they don't currently exist), the appropriate webhooks in dockerhub that will automatically sync branches upon promotion of `latest` -> `staging` -> `production` docker images._

4. You're ready to go!


