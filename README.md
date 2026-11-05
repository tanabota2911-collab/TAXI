import React, { useState, useEffect } from 'react';
import { StyleSheet, Text, View, TouchableOpacity, ScrollView, Alert, ActivityIndicator } from 'react-native';
import * as Location from 'expo-location';

export default function App() {
  const [locationPermission, setLocationPermission] = useState(null);
  const [loading, setLoading] = useState(false);
  
  // 現在の運行ステータス ('idle': 空車, 'riding': 実車中)
  const [status, setStatus] = useState('idle');
  
  // 現在記録中のデータ
  const [currentRide, setCurrentRide] = useState({
    pickupTime: null,
    pickupLocation: null,
    pickupAddress: null,
  });

  // 過去の運行ログの蓄積（ここに数ヶ月分を溜める、またはDify等のAPIに送信する）
  const [rideLogs, setRideLogs] = useState([]);

  // アプリ起動時に位置情報の権限を要求
  useEffect(() => {
    (async () => {
      let { status } = await Location.requestForegroundPermissionsAsync();
      setLocationPermission(status === 'granted');
    })();
  }, []);

  // 現在地と住所を取得する共通関数
  const getCurrentLocationData = async () => {
    setLoading(true);
    try {
      // 高精度で現在地を取得
      let loc = await Location.getCurrentPositionAsync({
        accuracy: Location.Accuracy.High,
      });
      
      const { latitude, longitude } = loc.coords;
      let addressText = '住所を取得できませんでした';

      // 緯度経度から住所（地名）に変換（逆ジオコーディング）
      let geocode = await Location.reverseGeocodeAsync({ latitude, longitude });
      if (geocode && geocode.length > 0) {
        const place = geocode[0];
        // 日本の住所形式に合わせて組み立て (例: 京都府亀岡市追分町...)
        addressText = `${place.region || ''}${place.city || ''}${place.street || ''}${place.name || ''}`;
      }

      setLoading(false);
      return {
        coords: { latitude, longitude },
        address: addressText,
        time: new Date().toLocaleString('ja-JP'),
      };
    } catch (error) {
      setLoading(false);
      Alert.alert('エラー', '位置情報の取得に失敗しました。GPSを確認してください。');
      return null;
    }
  };

  // 【乗車ボタン】を押したときの処理
  const handlePickup = async () => {
    if (!locationPermission) {
      Alert.alert('権限エラー', '位置情報の利用が許可されていません。');
      return;
    }

    const data = await getCurrentLocationData();
    if (data) {
      setCurrentRide({
        pickupTime: data.time,
        pickupLocation: data.coords,
        pickupAddress: data.address,
      });
      setStatus('riding');
    }
  };

  // 【降車ボタン】を押したときの処理
  const handleDropoff = async () => {
    if (!locationPermission) return;

    const data = await getCurrentLocationData();
    if (data) {
      const completedRide = {
        id: Date.now().toString(),
        pickupTime: currentRide.pickupTime,
        pickupAddress: currentRide.pickupAddress,
        pickupLocation: currentRide.pickupLocation,
        dropoffTime: data.time,
        dropoffAddress: data.address,
        dropoffLocation: data.coords,
      };

      // ログに保存（将来的にここでDifyのAPI等にPOST送信するコードを挟めます）
      setRideLogs([completedRide, ...rideLogs]);
      
      // ステータスをリセット
      setCurrentRide({ pickupTime: null, pickupLocation: null, pickupAddress: null });
      setStatus('idle');
      
      Alert.alert('記録完了', `乗車地: ${completedRide.pickupAddress}\n降車地: ${completedRide.dropoffAddress}\nデータを記録しました。`);
    }
  };

  return (
    <View style={styles.container}>
      {/* ヘッダー */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>タクシー運行記録エージェント</Text>
      </View>

      {/* メイン操作エリア */}
      <View style={styles.mainArea}>
        {loading ? (
          <View style={styles.loadingContainer}>
            <ActivityIndicator size="large" color="#007AFF" />
            <Text style={styles.loadingText}>位置情報を取得中...</Text>
          </View>
        ) : (
          <View style={styles.statusBox}>
            <Text style={styles.statusLabel}>現在の状態</Text>
            <Text style={[styles.statusText, status === 'riding' ? styles.statusRiding : styles.statusIdle]}>
              {status === 'riding' ? '🔴 実車中 (乗客あり)' : '🟢 空車中 (乗客待ち)'}
            </Text>
            
            {status === 'riding' && (
              <View style={styles.currentInfo}>
                <Text style={styles.infoText}>⏱ 乗車時刻: {currentRide.pickupTime}</Text>
                <Text style={styles.infoText}>📍 乗車場所: {currentRide.pickupAddress}</Text>
              </View>
            )}
          </View>
        )}

        {/* 操作ボタン */}
        <View style={styles.buttonContainer}>
          {status === 'idle' ? (
            <TouchableOpacity 
              style={[styles.actionButton, styles.pickupButton, loading && styles.disabledButton]} 
              onPress={handlePickup}
              disabled={loading}
            >
              <Text style={styles.buttonText}>乗車記録スタート</Text>
            </TouchableOpacity>
          ) : (
            <TouchableOpacity 
              style={[styles.actionButton, styles.dropoffButton, loading && styles.disabledButton]} 
              onPress={handleDropoff}
              disabled={loading}
            >
              <Text style={styles.buttonText}>降車記録完了</Text>
            </TouchableOpacity>
          )}
        </View>
      </View>

      {/* 履歴エリア */}
      <View style={styles.historyArea}>
        <Text style={styles.historyTitle}>本日の運行ログ ({rideLogs.length}件)</Text>
        <ScrollView style={styles.logList}>
          {rideLogs.map((log) => (
            <View key={log.id} style={styles.logCard}>
              <Text style={styles.logTime}>乗車: {log.pickupTime} → 降車: {log.dropoffTime.split(' ')[1]}</Text>
              <Text style={styles.logAddress} numberOfLines={1}>🛫 {log.pickupAddress}</Text>
              <Text style={styles.logAddress} numberOfLines={1}>🛬 {log.dropoffAddress}</Text>
            </View>
          ))}
          {rideLogs.length === 0 && (
            <Text style={styles.emptyText}>まだ記録されたデータはありません</Text>
          )}
        </ScrollView>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#F5F7FA',
    paddingTop: 50,
  },
  header: {
    padding: 20,
    backgroundColor: '#1E293B',
    alignItems: 'center',
  },
  headerTitle: {
    color: '#FFF',
    fontSize: 20,
    fontWeight: 'bold',
  },
  mainArea: {
    padding: 20,
    alignItems: 'center',
  },
  statusBox: {
    width: '100%',
    backgroundColor: '#FFF',
    padding: 20,
    borderRadius: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
    marginBottom: 20,
  },
  statusLabel: {
    fontSize: 14,
    color: '#64748B',
    marginBottom: 5,
  },
  statusText: {
    fontSize: 22,
    fontWeight: 'bold',
    marginBottom: 10,
  },
  statusIdle: {
    color: '#10B981',
  },
  statusRiding: {
    color: '#EF4444',
  },
  currentInfo: {
    borderTopWidth: 1,
    borderTopColor: '#E2E8F0',
    paddingTop: 10,
    marginTop: 5,
  },
  infoText: {
    fontSize: 14,
    color: '#334155',
    marginTop: 4,
  },
  loadingContainer: {
    height: 120,
    justifyContent: 'center',
    alignItems: 'center',
  },
  loadingText: {
    marginTop: 10,
    color: '#007AFF',
  },
  buttonContainer: {
    width: '100%',
  },
  actionButton: {
    width: '100%',
    height: 75,
    borderRadius: 16,
    justifyContent: 'center',
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.2,
    shadowRadius: 5,
    elevation: 5,
  },
  pickupButton: {
    backgroundColor: '#007AFF',
  },
  dropoffButton: {
    backgroundColor: '#10B981',
  },
  disabledButton: {
    backgroundColor: '#CBD5E1',
  },
  buttonText: {
    color: '#FFF',
    fontSize: 22,
    fontWeight: 'bold',
  },
  historyArea: {
    flex: 1,
    padding: 20,
    backgroundColor: '#FFF',
    borderTopLeftRadius: 24,
    borderTopRightRadius: 24,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: -3 },
    shadowOpacity: 0.05,
    shadowRadius: 5,
    elevation: 5,
  },
  historyTitle: {
    fontSize: 16,
    fontWeight: 'bold',
    color: '#1E293B',
    marginBottom: 15,
  },
  logList: {
    flex: 1,
  },
  logCard: {
    backgroundColor: '#F8FAFC',
    padding: 12,
    borderRadius: 8,
    marginBottom: 10,
    borderLeftWidth: 4,
    borderLeftColor: '#3B82F6',
  },
  logTime: {
    fontSize: 12,
    color: '#64748B',
    fontWeight: '600',
    marginBottom: 4,
  },
  logAddress: {
    fontSize: 14,
    color: '#1E293B',
    marginTop: 2,
  },
  emptyText: {
    textAlign: 'center',
    color: '#94A3B8',
    marginTop: 30,
  },
});
