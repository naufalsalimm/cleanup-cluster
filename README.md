#!/bin/bash

echo "🧹 Mulai proses bersih-bersih cluster (Pod, ReplicaSet, Deployment)..."

# Jumlah maksimal objek yang dihapus tiap batch
BATCH_LIMIT=50

delete_pods_by_status() {
  STATUS=$1
  echo "➡️  Menghapus pod dengan status: $STATUS (max $BATCH_LIMIT)..."
  kubectl get pods --all-namespaces | grep "$STATUS" | \
    awk '{print $1, $2}' | head -n $BATCH_LIMIT | while read ns name; do
      echo "   🗑️  Hapus pod: $name (ns: $ns)"
      kubectl delete pod -n "$ns" "$name" --grace-period=0 --force
    done
}

# Hapus pod dengan status Completed (alias Succeeded)
echo "➡️  Menghapus pod status: Completed/Succeeded (max $BATCH_LIMIT)..."
kubectl get pods --all-namespaces --field-selector=status.phase=Succeeded \
  -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name" --no-headers | \
  head -n $BATCH_LIMIT | while read ns name; do
    echo "   🗑️  Hapus pod: $name (ns: $ns)"
    kubectl delete pod -n "$ns" "$name" --grace-period=0 --force
  done

# Hapus pod error lainnya
delete_pods_by_status "Error"
delete_pods_by_status "CrashLoopBackOff"
delete_pods_by_status "ImagePullBackOff"
delete_pods_by_status "ContainerStatusUnknown"

# Mendapatkan daftar pod yang memiliki status "Evicted"
evicted_pods=$(kubectl get pods --all-namespaces --field-selector=status.phase=Failed -o custom-columns=":metadata.namespace,:metadata.name,:status.reason" | grep Evicted)

# Memeriksa apakah ada pod yang evicted
if [ -z "$evicted_pods" ]; then
    echo "Tidak ada pod yang evicted."
else
    # Melakukan penghapusan untuk setiap pod yang evicted
    echo "$evicted_pods" | while read namespace pod reason; do
        echo "Menghapus pod $pod dari namespace $namespace yang berstatus $reason"
        kubectl delete pod "$pod" -n "$namespace"
    done
fi

# Hapus ReplicaSet tanpa pod aktif
echo "➡️  Menghapus ReplicaSet tanpa pod..."
kubectl get rs --all-namespaces --no-headers | awk '$3 == 0 && $4 == 0 {print $1, $2}' | while read ns rsname; do
  echo "   🗑️  Hapus ReplicaSet: $rsname (ns: $ns)"
  kubectl delete rs -n "$ns" "$rsname"
done

# Hapus Deployment tanpa pod
echo "➡️  Menghapus Deployment tanpa pod..."
kubectl get deploy --all-namespaces --no-headers | while read ns name ready up-to-date available age; do
  REPLICAS=$(kubectl get deploy -n "$ns" "$name" -o jsonpath='{.status.replicas}')
  [[ "$REPLICAS" == "0" || -z "$REPLICAS" ]] && {
    echo "   🗑️  Hapus Deployment: $name (ns: $ns)"
    kubectl delete deploy -n "$ns" "$name"
  }
done

echo "✅ Bersih-bersih selesai! Jalankan ulang kalau masih ada sisa batch lain."
