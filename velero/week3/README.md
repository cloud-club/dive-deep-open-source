# Week 3 - Velero 코드 분석

### velero backup create 

명령어 진입
~~~go
	c := &cobra.Command{
		Use:   use + " NAME",
		Short: "Create a backup",
		Args:  cobra.MaximumNArgs(1),
		Run: func(c *cobra.Command, args []string) {
			cmd.CheckError(o.Complete(args, f))
			cmd.CheckError(o.Validate(c, args, f))
			cmd.CheckError(o.Run(c, f))
		},
~~~


백업 객체 구성
~~~go
func (o *CreateOptions) Run(c *cobra.Command, f client.Factory) error {
	backup, err := o.BuildBackup(f.Namespace())
	if err != nil {
		return err
	}

	if printed, err := output.PrintWithFormat(c, backup); printed || err != nil {
		return err
	}
~~~

controller-runtime 클라이언트를 통해 k8s api 호출
~~~go
	err = o.client.Create(context.TODO(), backup, &kbclient.CreateOptions{})
	if err != nil {
		return err
	}

	fmt.Printf("Backup request %q submitted successfully.\n", backup.Name)
~~~

k8s 클라이언트 구성
~~~go
func (f *factory) KubebuilderClient() (kbclient.Client, error) {
	clientConfig, err := f.ClientConfig()
	if err != nil {
		return nil, err
	}

	scheme := runtime.NewScheme()
	if err := velerov1api.AddToScheme(scheme); err != nil {
		return nil, err
	}
	if err := velerov2alpha1api.AddToScheme(scheme); err != nil {
		return nil, err
	}
	if err := k8scheme.AddToScheme(scheme); err != nil {
		return nil, err
	}
	if err := apiextv1beta1.AddToScheme(scheme); err != nil {
		return nil, err
	}
	if err := apiextv1.AddToScheme(scheme); err != nil {
		return nil, err
	}
	if err := snapshotv1api.AddToScheme(scheme); err != nil {
		return nil, err
	}
	kubebuilderClient, err := kbclient.New(clientConfig, kbclient.Options{
		Scheme: scheme,
	})

	if err != nil {
		return nil, err
	}

	return kubebuilderClient, nil
}
~~~

백업 객체 빌드
~~~go
// ForBackup is the constructor for a BackupBuilder.
func ForBackup(ns, name string) *BackupBuilder {
	return &BackupBuilder{
		object: &velerov1api.Backup{
			TypeMeta: metav1.TypeMeta{
				APIVersion: velerov1api.SchemeGroupVersion.String(),
				Kind:       "Backup",
			},
			ObjectMeta: metav1.ObjectMeta{
				Namespace: ns,
				Name:      name,
			},
		},
	}
}
~~~

http 요청을 통한 etcd에 저장
~~~http
POST /apis/velero.io/v1/namespaces/{namespace}/backups
Content-Type: application/json

{
  "apiVersion": "velero.io/v1",
  "kind": "Backup",
  "metadata": {
    "name": "backup-name",
    "namespace": "velero"
  },
  "spec": {
    "includedNamespaces": ["*"],
    "ttl": "720h0m0s"
    // ... 기타 백업 설정
  }
}
~~~

백업 컨트롤러가 이를 감지하고 실제 백업 프로세스를 시작
~~~go
// pkg/controller/backup_controller.go에서 실행되는 로직
func (r *BackupReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 백업 객체를 etcd에서 읽어옴
    backup := &velerov1api.Backup{}
    if err := r.Get(ctx, req.NamespacedName, backup); err != nil {
        // ...
    }
    
    // 백업 프로세스 시작
    // ...
}
~~~

> 요약
1. CLI 명령어: velero create backup 실행
2. 백업 객체 생성: BuildBackup()으로 Velero Backup CRD 객체 구성
3. Kubernetes 클라이언트: controller-runtime 클라이언트를 통해 API 호출
4. HTTP 요청: Kubernetes API 서버에 POST 요청
5. API 서버 처리: 인증, 검증, Admission Controller 처리
6. etcd 저장: /registry/velero.io/backups/{namespace}/{backup-name} 키로 저장
7. 컨트롤러 감지: Velero 백업 컨트롤러가 변경사항을 감지하고 실제 백업 작업 시작


## Next Week
https://github.com/vmware-tanzu/helm-charts/issues/628

기여 요청