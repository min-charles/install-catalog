
# CatalogContoller 설치 가이드

## 구성 요소 및 버전

## Prerequisites

## 폐쇄망 구축 가이드

1. 사용하는 image를 다운받고 저장합니다.

   - 작업 디렉토리 생성 및 환경 설정

   ```bash
   mkdir -p ~/catalog-controller-install
   export CATALOG_HOME=~/catalog-controller-install
   export CATALOG_VERSION=0.3.0
   cd $CATALOG_HOME
   ```

   - 외부 네트워크 통신이 가능한 환경에서 필요한 이미지를 다운받습니다.

   ```bash
   # TEMPLATE SERVICE BROKER 이미지 Pull
   docker pull quay.io/kubernetes-service-catalog/service-catalog:v${CATALOG_VERSION}

   # 이미지 Save
   docker save quay.io/kubernetes-service-catalog/service-catalog:v${CATALOG_VERSION} > service-catalog_v${CATALOG_VERSION}.tar
   ```

2. 폐쇄망으로 이미지 파일(.tar)을 옮깁니다.

3. 폐쇄망에서 사용하는 image repository에 이미지를 push 합니다.

   ```bash
   # 이미지 레지스트리 주소
   REGISTRY=[IP:PORT]

   # 이미지 Load
   docker load < service-catalog_v${CATALOG_VERSION}.tar

   # 이미지 Tag
   docker tag quay.io/kubernetes-service-catalog/service-catalog:v${CATALOG_VERSION} ${REGISTRY}/quay.io/kubernetes-service-catalog/service-catalog:v${CATALOG_VERSION}

   # 이미지 Push
   docker push ${REGISTRY}/quay.io/kubernetes-service-catalog/service-catalog:v${CATALOG_VERSION}
   ```

## 설치 가이드
1. [설치에 필요한 crd 생성](#Step-1-설치에-필요한-crd-생성)
2. [catalog controller namespace 및 servcice account 생성](#Step-2-catalog-controller-namespace-및-servcice-account-생성)
3. [catalog manager 생성](#Step-3-catalog-manager-생성)
4. [webhook 인증 키 생성](#Step-4-webhook-인증-키-생성)
5. [catalog-webhook 생성](#Step-5-catalog-webhook-생성)


## Step 1. 설치에 필요한 crd 생성
- 목적 : `CatalogController crd 생성`
- 생성 순서 : 아래 command로 yaml 적용
    - kubectl apply -f crds/ ([폴더](./manifest/crds/key-mapping)) 

## Step 2. catalog controller namespace 및 servcice account 생성
- 목적 : `catalog controller namespace 및 servcice account 생성`
- 생성 순서 : 아래 command로 yaml 적용
    - kubectl create namespace catalog
    - kubectl apply -f serviceaccounts.yaml ([파일](./manifest/serviceaccounts.yaml))
    - kubectl apply -f rbac.yaml ([파일](./manifest/rbac.yaml))

## Step 3. catalog manager 생성
- 목적 : `catalog manager 생성`
- 생성 순서 : 아래 command로 yaml 적용
    - kubectl apply -f controller-manager-deployment.yaml ([파일](./manifest/controller-manager-deployment.yaml))
        - 비고: 각 파일에 image 항목이 있는 경우, registry주소의 이미지를 사용해야 합니다. (${REGISTRY}/quay.io/kubernetes-service-catalog/service-catalog:v${CATALOG_VERSION})
    - kubectl apply -f controller-manager-service.yaml ([파일](./manifest/controller-manager-service.yaml))

## Step 4. webhook 인증 키 생성
- 목적 : `webhook 인증에 필요한 키를 생성`
- 생성 순서 : 아래 command 적용
    - openssl genrsa -out rootca.key 2048
    - openssl req -x509 -new -nodes -key rootca.key -sha256 -days 3650 -subj /C=KO/ST=None/L=None/O=None/CN=catalog-catalog-webhook -out rootca.crt
        - 비고: .rnd 파일이 없어서 해당 명령어 실행이 안되는 경우, /etc/ssl/openssl.cnf 파일의  "RANDFILE = $ENV::HOME/.rnd" 부분을 주석처리 합니다.
    - v3.ext 파일 생성 ([파일](./manifest/ca/v3.ext))
    - openssl req -new -newkey rsa:2048 -sha256 -nodes -keyout server.key -subj /C=KO/ST=None/L=None/O=None/CN=catalog-catalog-webhook -out server.csr
    - openssl x509 -req -in server.csr -CA rootca.crt -CAkey rootca.key -CAcreateserial -out server.crt -days 3650 -sha256 -extfile ./v3.ext
    - openssl base64 -in rootca.crt -out key0
    - openssl base64 -in server.crt -out cert
    - openssl base64 -in server.key -out key

## Step 5. catalog-webhook 생성
- 목적 : `catalog-webhook 생성`
- 생성 순서 : 아래 command 적용
    - kubectl apply -f webhook-register.yaml ([파일](./manifest/webhook-register.yaml))
        - 비고: 해당 파일의 변수를 7번에서 생성한 키로 대체 합니다. (단, 생성된 키 공백은 모두 지우셔야 합니다.)
            - {{ b64enc $ca.Cert }} --> key0 내부 값으로 대체
            - {{ b64enc $cert.Cert }} --> cert 내부 값으로 대체
            - {{ b64enc $cert.Key }} --> key 내부 값으로 대체
    - kubectl apply -f webhook-deployment.yaml ([파일](./manifest/webhook-deployment.yaml))
        - 비고: 각 파일에 image 항목이 있는 경우, registry주소의 이미지를 사용해야 합니다. (${REGISTRY}/quay.io/kubernetes-service-catalog/service-catalog:v${CATALOG_VERSION})
    - kubectl apply -f webhook-service.yaml ([파일](./manifest/webhook-service.yaml))

## 삭제 가이드
1. [사용중인 리소스 제거](#Step-1-사용중인-리소스-제거)
2. [설치 리소스 제거](#Step-2-설치-리소스-제거)
3. [CRD 제거](#Step-3-CRD-제거)

## Step 1. 사용중인 리소스 제거
- 목적 : `사용중인 리소스 차례로 제거`
- 삭제 순서 : 아래 command 순서대로 적용
    - kubectl delete servicebinding --all --all-namespaces
    - kubectl delete serviceinstance --all --all-namespaces
    - kubectl delete servicebinding --all --all-namespaces
    - kubectl delete clusterservicebroker --all --all-namespaces
    - kubectl delete servicebroker --all --all-namespaces

## Step 2. 설치 리소스 제거
- 목적 : `설치에 필요한 pod, service 등의 리소스 제거`
- 삭제 순서 : 설치에 진행했던 yaml 파일 역순으로 적용
    - kubectl delete -f webhook-service.yaml ([파일](./manifest/webhook-service.yaml))
    - kubectl delete -f webhook-deployment.yaml ([파일](./manifest/webhook-deployment.yaml))
    - kubectl delete -f webhook-register.yaml ([파일](./manifest/webhook-register.yaml))
    - kubectl delete -f controller-manager-service.yaml ([파일](./manifest/controller-manager-service.yaml))
    - kubectl delete -f controller-manager-deployment.yaml ([파일](./manifest/controller-manager-deployment.yaml))
    - kubectl delete -f rbac.yaml ([파일](./manifest/rbac.yaml))
    - kubectl delete -f serviceaccounts.yaml ([파일](./manifest/serviceaccounts.yaml))
    - kubectl delete namespace catalog

## Step 3. CRD 제거
- 목적 : `CRD 제거`
- 삭제 순서 : 아래 command로 yaml 적용
    - kubectl delete -f crds/ ([폴더](./manifest/crds/key-mapping)) 
