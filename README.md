# uc2-configurator

This currently grabs all config and CAD files from my fork (AlecVercruysse/UC2-GIT) since each module and application needs it's own config, and upstream does not yet have configuration files written. Ideally, this will change to bionanoimaging/UC2-GIT.

See the Issues tab for open enhancements or bugs that need to be fixed (contributions welcome!). There's a lot that can be implemented with minimal effort, including ignoring stl file versioning in the filename (e.g. when a config includes `file.stl`, `file_v2.stl` matches), and including a commit number in an application config to require a specific version of an stl.

This repo can also be ran locally:
Note that in the netlify app, everything is still client-side. 

## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```
