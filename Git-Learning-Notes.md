

- ## git remote: warning: Large files detected. 

remote: error: GH001: Large files detected. You may want to try Git Large File Storage - https://git-lfs.github.com.
remote: error: File .sus-sm-rdp-core_mta_build_tmp/sus-sm-rdp-core-mtx-sidecar/data.zip is 108.53 MB; this exceeds GitHub Enterprise's file size limit of 100.00 MB
To github.wdf.sap.corp:smrdp/smrdp-core.git
``` 
git filter-branch -f --prune-empty --index-filter "git rm -rf --cached --ignore-unmatch .sus-sm-rdp-core_mta_build_tmp/sus-sm-rdp-core-mtx-sidecar/data.zip" --tag-name-filter cat -- --all
``` 
