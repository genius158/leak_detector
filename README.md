# leak_detector

find vm service
```
Service.getInfo().then((serviceProtocolInfo) {
      Uri uri = serviceProtocolInfo.serverUri!;
      uri = uri.replace(host: host);

      String url = convertToWebSocketUrl(serviceProtocolUrl: uri).toString();
      vmServiceConnectUri(url).then((VmService vmService) async {
        _vmService = vmService;
      });
    });
```

_mainIsolateID
```
 List<IsolateRef> isolates = await _vmService.getVM().isolates!;
 late IsolateRef ref;

 for (var i = 0; i < isolates.length; ++i) {
   IsolateRef t = isolates[i];
   if(t.name!.contains('main')) {
     ref = t;
     break;
   }
 }
 _mainIsolateID = ref.id!;
```
_libraryID
```
 final Isolate isolate = await _vmService.getIsolate(_mainIsolateID);
 var libraries = isolate.libraries!;
 for (LibraryRef ref in libraries) {
   // 顶级方法所在的Dart文件
   if(ref.uri!.endsWith('expando_memory.dart')) {
     _libraryID = ref.id!;
   }
 }
```

getExpando in expando_memory.dart
```
Expando getExpando() {
  return expandoIns;
}
```

_expandoInstance
```
 InstanceRef valueRef = await _vmService.invoke(
        _mainIsolateID,
        _libraryID,
        "getExpando",
        []
 ) as InstanceRef;
 // 这里的 id 就是 obj 对应的 id
 String? objectId = valueRef.id;
 var value = await _vmService.getObject(_mainIsolateID, objectId!);
 _expandoInstance value as Instance?;
```
_dataInstance(expando data instance)
```
 late BoundField _dataField;
 for (BoundField field in _expandoInstance.fields!) {
   if (field.decl!.name == "_data") {
     _dataField = field;
     break;
   }
 }

 InstanceRef instanceRef = _dataField.value;
 Instance? _dataInstance = await _vmService.getObject(_mainIsolateID, instanceRef.id!) as Instance?;
```

GC
```
 Future<void> gc() async {
   // String _collectAllGarbageMethodName = '_collectAllGarbage';
   // await _vmService.callMethod(_collectAllGarbageMethodName, isolateId: _mainIsolateID);
   await _vmService.getAllocationProfile(isolate.id!, gc: true);
 }

```

check Leak
```
 gc();
 List instanceRefs = _dataInstance.elements!;
 for (InstanceRef? instanceRef in instanceRefs) {
   if (instanceRef != null) {
     var instance = await _vmService.getObject(_mainIsolateID, instanceRef.id) as Instance?;
     InstanceRef propertyValue = instance!.propertyValue!;
     if(propertyValue != null) {
      // leak check
     }
   }
 }
```