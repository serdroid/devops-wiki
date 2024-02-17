# Transfer Nomad Variables Between Clusters

## Export Variables
This is a sample output of a nomad variable. CreateIndex, ModifyIndex, CreateTime, and ModifyTime lines should be removed because they could clash with existing values on the target cluster.
```
nomad var get -out json -namespace=test nomad/jobs/foo
{
  "Namespace": "test",
  "Path": "nomad/jobs/foo",
  "CreateIndex": 1204,
  "ModifyIndex": 1204,
  "CreateTime": 1701111484705586994,
  "ModifyTime": 1701111484705586994,
  "Items": {
    "FOO_USER": "fooser",
    "FOO_PASSWORD": "password"
  }
}
```

Create a directory to export all variables. Loop over the variables list, get the variable definition as json, delete the lines that contains _"Create"_ and _"Modify"_, replace source namespace with target, and redirect output to a variable named file.

```
cd /tmp
mkdir vars
SOURCENS=my-test
TARGETNS=my-stage
nomad var list -out table -namespace=$SOURCENS | grep -v Namesp | awk '{print $2}' | while read vn; do nomad var get -out json -namespace=$SOURCENS $vn | sed -e '/Create/d' -e '/Modify/d' -e "/Namespace/ s/$SOURCENS/$TARGETNS/" > vars/${vn##*/}; done
```

Create a tar archive and transfer it into new cluster. 
```
tar -czf vars.tgz vars
scp -p vars.tgz someone@my-stage.example.com:/tmp/
```


## Create Variables On The New Cluster
Extract the vars tar archive and create the variables with below commands.
```
tar -xf vars.tgz

ls -1 vars/* | while read fl
do
nomad var put - <<< $(cat $fl)
done
```


## Purge All Variables In A Namespace
Sometimes it is needed to purge all variables in a namespace. Below command could be used to remove all variables in a namespace.

```
TARGETNS=my-stage
nomad var list -out table -namespace=$TARGETNS | grep -v Namesp | awk '{print $2}' | while read vn
do
 nomad var purge -namespace=$TARGETNS $vn
done
```

