### 바인드 마운트
- 호스트의 스토리지에 직접적으로 연결하여 접근할때 사용하게 됩니다.
- 바인트 마운트는 호스트 컴퓨터 파일 시스템의 디렉터리를 컨테이너 파일시스템의 디렉터리로 만듭니다. 그래서 볼륨과 마찬가지로 컨테이너의 입장에서는 그냥 평범한 디렉터리와 똑같습니다.즉, 호스트 컴퓨터에서 접근한 파일 시스템이라면 무엇이든 컨테이너에서도 접근해서 사용할 수 있습니다.

- 관리자 권한이 필요한 경우, `USER` 인스트럭션을 사용해서 권한을 넘겨 받을 수 있게 됩니다. 
결론적으로 컨테이너에서 호스트로, 호스트에서 컨테이너로 접근이 가능하게 됩니다.

- 그렇다고 바이트 마운트가 만능으로 호스트 컴퓨터의 파일 시스템을 원화할게 사용하는 것은 아닙니다.
- 다음 3가지 경우를 주의해야합니다.
1. 컨테이너의 마운트 대상 디렉터리가 이미 존재하고 이미지 레이어에 이 디렉터리의 파일이 포함되어있다면,이미 존재하는 디렉터리의 파일들이 마운트 되는 디렉터리에 의해 모두 대체되게 됩니다.
2. 하지만 운영체제에 따라 파일의 경우에는 해당 디렉터리에 추가되는 형태로 나타나게 됩니다.
3. 분산 파일 스토리지(AWS S3 / SMB / Azure Files)를 컨테이너에 마운트하면 일반적인 파일 시스템의 일부처럼 보이기는 하겠지만 지원하지 않는 동작이 있을 수 있습니다.
- 분산 파일 스토리지에 컨테이너를 만들게 되면, 이러한 상황이 발생할 수 있다는 것을 주의해야하고, 바인드 마운트를 사용해서 발생하는 버그 상황은 실행해보지 않지 않고 알기 쉽지 않습니다.
