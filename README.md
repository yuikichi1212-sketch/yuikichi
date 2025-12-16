import React, { useState, useEffect, useRef } from "react";
import { base44 } from "@/api/base44Client";
import { Button } from "@/components/ui/button";
import { Card } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { 
  Navigation, 
  Mic, 
  MicOff, 
  Search,
  MapPin,
  Clock,
  Car,
  Train,
  Bike,
  Footprints,
  Loader2,
  Send,
  X,
  MessageSquare,
  ChevronDown,
  ChevronUp,
  Maximize2,
  Minimize2,
  AlertTriangle
} from "lucide-react";
import { motion, AnimatePresence } from "framer-motion";
import NavMap from "../components/nav/NavMap";

export default function YuikichiNav() {
  const [currentLocation, setCurrentLocation] = useState(null);
  const [destination, setDestination] = useState(null);
  const [searchQuery, setSearchQuery] = useState("");
  const [isSearching, setIsSearching] = useState(false);
  const [isListening, setIsListening] = useState(false);
  const [transcript, setTranscript] = useState("");
  const [transportMode, setTransportMode] = useState("driving");
  const [routeInfo, setRouteInfo] = useState(null);
  const [messages, setMessages] = useState([
    { role: "assistant", content: "こんにちは！ゆいきちナビです。「〇〇まで行きたい」や「今どこ？」など、何でも話しかけてください！" }
  ]);
  const [isAIThinking, setIsAIThinking] = useState(false);
  const [showChat, setShowChat] = useState(false);
  const [textInput, setTextInput] = useState("");
  const [navMode, setNavMode] = useState(false);
  const [routeCoordinates, setRouteCoordinates] = useState(null);
  const [isCalculatingRoute, setIsCalculatingRoute] = useState(false);
  const [trafficInfo, setTrafficInfo] = useState(null);
  const [passedCoordinates, setPassedCoordinates] = useState([]);
  const [settings, setSettings] = useState({
    routeColor: "#3b82f6",
    passedRouteColor: "#94a3b8",
    routeWidth: 8,
    autoReroute: true,
    rerouteThreshold: 50
  });
  const [showSettings, setShowSettings] = useState(false);
  const [transitDetails, setTransitDetails] = useState(null);
  const [followMode, setFollowMode] = useState(true);
  const [showRouteDetails, setShowRouteDetails] = useState(false);
  const [currentInstruction, setCurrentInstruction] = useState(null);
  const [nextInstruction, setNextInstruction] = useState(null);
  const [distanceToNext, setDistanceToNext] = useState(null);
  
  const recognitionRef = useRef(null);
  const messagesEndRef = useRef(null);
  const lastAnnouncedInstructionRef = useRef(null);

  useEffect(() => {
    getCurrentLocation();
    initializeSpeechRecognition();
  }, []);

  useEffect(() => {
    if (messagesEndRef.current) {
      messagesEndRef.current.scrollIntoView({ behavior: "smooth" });
    }
  }, [messages]);

  useEffect(() => {
    if (currentLocation && destination) {
      calculateRoute();
    }
  }, [currentLocation, destination, transportMode]);

  useEffect(() => {
    // リアルタイムで通過済みルートを更新
    if (navMode && currentLocation && routeCoordinates && routeCoordinates.length > 0) {
      updatePassedRoute();
      checkReroute();
      updateNavInstructions();
    }
  }, [currentLocation, navMode]);

  const updateNavInstructions = () => {
    if (!routeInfo?.instructions || routeInfo.instructions.length === 0) return;

    // 現在地に最も近い案内ステップを見つける
    let minDistance = Infinity;
    let nearestIndex = 0;

    routeInfo.instructions.forEach((instruction, idx) => {
      // 簡易的に案内の位置を推定（実際はルート上の座標を使う）
      const distance = Math.abs(idx - passedCoordinates.length / 10);
      if (distance < minDistance) {
        minDistance = distance;
        nearestIndex = idx;
      }
    });

    const current = routeInfo.instructions[nearestIndex];
    const next = routeInfo.instructions[nearestIndex + 1];

    setCurrentInstruction(current);
    setNextInstruction(next);

    if (current) {
      const dist = Math.round(current.distance);
      setDistanceToNext(dist);

      // 200m以内で音声案内
      if (dist < 200 && lastAnnouncedInstructionRef.current !== nearestIndex) {
        announceInstruction(current, dist);
        lastAnnouncedInstructionRef.current = nearestIndex;
      }
    }
  };

  const announceInstruction = (instruction, distance) => {
    if (!('speechSynthesis' in window)) return;

    const text = `${distance}メートル先、${instruction.instruction}`;
    
    const utterance = new SpeechSynthesisUtterance(text);
    utterance.lang = 'ja-JP';
    utterance.rate = 1.0;
    utterance.pitch = 1.0;
    utterance.volume = 1.0;
    
    window.speechSynthesis.speak(utterance);
    
    // メッセージにも追加
    setMessages(prev => [...prev, {
      role: "assistant",
      content: text
    }]);
  };

  const updatePassedRoute = () => {
    if (!routeCoordinates || routeCoordinates.length === 0) return;

    const passed = [];
    for (let i = 0; i < routeCoordinates.length; i++) {
      const coord = routeCoordinates[i];
      const distance = calculateDistance(
        currentLocation.lat,
        currentLocation.lng,
        coord[0],
        coord[1]
      );

      if (distance < 0.05) {
        passed.push(...routeCoordinates.slice(0, i + 1));
        setPassedCoordinates(passed);
        break;
      }
    }
  };

  const checkReroute = () => {
    if (!settings.autoReroute || !routeCoordinates || routeCoordinates.length === 0) return;

    let minDistance = Infinity;
    for (const coord of routeCoordinates) {
      const distance = calculateDistance(
        currentLocation.lat,
        currentLocation.lng,
        coord[0],
        coord[1]
      );
      minDistance = Math.min(minDistance, distance);
    }

    if (minDistance > settings.rerouteThreshold / 1000) {
      console.log("ルートから外れました。リルート中...");
      setMessages(prev => [...prev, {
        role: "assistant",
        content: "ルートから外れました。新しいルートを計算しています..."
      }]);
      calculateRoute();
    }
  };

  const calculateDistance = (lat1, lon1, lat2, lon2) => {
    const R = 6371;
    const dLat = toRad(lat2 - lat1);
    const dLon = toRad(lon2 - lon1);
    const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
              Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) *
              Math.sin(dLon/2) * Math.sin(dLon/2);
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    return R * c;
  };

  const getCurrentLocation = () => {
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(
        (position) => {
          setCurrentLocation({
            lat: position.coords.latitude,
            lng: position.coords.longitude,
            name: "現在地"
          });
        },
        (error) => {
          console.error("位置情報取得エラー:", error);
          setCurrentLocation({ lat: 35.6812, lng: 139.7671, name: "東京駅付近" });
        }
      );

      // リアルタイムで位置情報を更新
      navigator.geolocation.watchPosition(
        (position) => {
          setCurrentLocation({
            lat: position.coords.latitude,
            lng: position.coords.longitude,
            name: "現在地"
          });
        },
        null,
        { enableHighAccuracy: true, maximumAge: 10000 }
      );
    } else {
      setCurrentLocation({ lat: 35.6812, lng: 139.7671, name: "東京駅付近" });
    }
  };

  const initializeSpeechRecognition = () => {
    if ('webkitSpeechRecognition' in window || 'SpeechRecognition' in window) {
      const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
      recognitionRef.current = new SpeechRecognition();
      recognitionRef.current.lang = 'ja-JP';
      recognitionRef.current.continuous = false;
      recognitionRef.current.interimResults = true;

      recognitionRef.current.onresult = (event) => {
        const current = event.resultIndex;
        const transcriptText = event.results[current][0].transcript;
        setTranscript(transcriptText);

        if (event.results[current].isFinal) {
          handleVoiceCommand(transcriptText);
        }
      };

      recognitionRef.current.onerror = (event) => {
        console.error('音声認識エラー:', event.error);
        setIsListening(false);
      };

      recognitionRef.current.onend = () => {
        setIsListening(false);
        setTranscript("");
      };
    }
  };

  const toggleListening = () => {
    if (isListening) {
      recognitionRef.current?.stop();
      setIsListening(false);
    } else {
      try {
        recognitionRef.current?.start();
        setIsListening(true);
        setShowChat(true);
      } catch (error) {
        console.error('音声認識開始エラー:', error);
      }
    }
  };

  const handleVoiceCommand = async (text) => {
    setMessages(prev => [...prev, { role: "user", content: text }]);
    setIsAIThinking(true);

    try {
      const trafficContext = trafficInfo ? `\n交通情報: ${trafficInfo}` : "";
      
      const prompt = `あなたは「ゆいきちナビ」という音声ナビゲーションアシスタントです。
ユーザーの質問に親しみやすく、簡潔に答えてください。

現在の状況:
- 現在地: ${currentLocation ? `${currentLocation.name} (緯度${currentLocation.lat.toFixed(4)}, 経度${currentLocation.lng.toFixed(4)})` : "取得中"}
- 目的地: ${destination ? destination.name : "未設定"}
- 移動手段: ${transportMode === "driving" ? "車" : transportMode === "walking" ? "徒歩" : transportMode === "transit" ? "電車・バス" : "自転車"}
${routeInfo ? `- 所要時間: ${routeInfo.duration}分\n- 距離: ${routeInfo.distance}km\n- 到着予定: ${routeInfo.arrival}` : ""}${trafficContext}

ユーザーの発言: "${text}"

以下のような質問に対応してください:
1. 「今どこ？」「現在地は？」→ 現在地情報を教える
2. 「〇〇まで行きたい」「〇〇への行き方」→ 目的地として設定することを伝え、検索を促す
3. 「あと何分？」「いつ着く？」→ ルートがあれば到着時刻、なければ目的地設定を促す
4. 「電車で」「歩いて」→ 移動手段の変更を確認
5. 「近くの〇〇」→ 検索を提案
6. 「渋滞は？」「交通状況は？」→ 交通情報があれば伝える

回答は2-3文で簡潔に。フレンドリーな口調で。`;

      const response = await base44.integrations.Core.InvokeLLM({
        prompt: prompt
      });

      setMessages(prev => [...prev, { role: "assistant", content: response }]);

      // 音声で読み上げ
      if ('speechSynthesis' in window) {
        const utterance = new SpeechSynthesisUtterance(response);
        utterance.lang = 'ja-JP';
        utterance.rate = 1.1;
        window.speechSynthesis.speak(utterance);
      }

      // 場所を含む発言なら検索を試みる
      if (text.includes("まで") || text.includes("行きたい") || text.includes("行き方") || text.includes("近く")) {
        const place = extractPlaceName(text);
        if (place) {
          await searchPlace(place);
        }
      }

    } catch (error) {
      console.error("AI応答エラー:", error);
      setMessages(prev => [...prev, { 
        role: "assistant", 
        content: "申し訳ありません。もう一度お試しください。" 
      }]);
    } finally {
      setIsAIThinking(false);
    }
  };

  const extractPlaceName = (text) => {
    const patterns = [
      /(.+?)まで/,
      /(.+?)への/,
      /(.+?)に行/,
      /近くの(.+)/,
    ];

    for (const pattern of patterns) {
      const match = text.match(pattern);
      if (match) {
        return match[1].trim();
      }
    }
    return null;
  };

  const handleSearch = async (e) => {
    e?.preventDefault();
    if (!searchQuery.trim()) return;
    await searchPlace(searchQuery);
  };

  const searchPlace = async (query) => {
    setIsSearching(true);
    try {
      const response = await base44.integrations.Core.InvokeLLM({
        prompt: `「${query}」という場所の正確な位置情報を返してください。実在する具体的な場所の情報を返してください。日本国内の場所を優先してください。`,
        add_context_from_internet: true,
        response_json_schema: {
          type: "object",
          properties: {
            name: { type: "string" },
            lat: { type: "number" },
            lng: { type: "number" },
            address: { type: "string" }
          }
        }
      });

      if (response.lat && response.lng) {
        setDestination({
          name: response.name || query,
          lat: response.lat,
          lng: response.lng,
          address: response.address || ""
        });

        await base44.entities.SearchHistory.create({
          query: query,
          transport_mode: transportMode
        });

        setSearchQuery("");
      } else {
        setMessages(prev => [...prev, {
          role: "assistant",
          content: `申し訳ありません。「${query}」の場所が見つかりませんでした。別の検索ワードをお試しください。`
        }]);
      }
    } catch (error) {
      console.error("検索エラー:", error);
      setMessages(prev => [...prev, {
        role: "assistant",
        content: "検索中にエラーが発生しました。もう一度お試しください。"
      }]);
    } finally {
      setIsSearching(false);
    }
  };

  const calculateRoute = async () => {
    if (!currentLocation || !destination) return;

    setIsCalculatingRoute(true);
    setPassedCoordinates([]);

    try {
      if (transportMode === "transit") {
        await calculateTransitRoute();
      } else {
        await calculateDirectRoute();
      }
    } catch (error) {
      console.error("ルート計算エラー:", error);
      const simpleRoute = generateSimpleRouteSync();
      setRouteCoordinates(simpleRoute);
    } finally {
      setIsCalculatingRoute(false);
    }
  };

  const calculateDirectRoute = async () => {
    const profile = {
      driving: "driving-car",
      walking: "foot-walking",
      cycling: "cycling-regular"
    }[transportMode] || "driving-car";

    const orsUrl = `https://api.openrouteservice.org/v2/directions/${profile}/geojson`;
    
    const requestBody = {
      coordinates: [
        [currentLocation.lng, currentLocation.lat],
        [destination.lng, destination.lat]
      ],
      preference: "recommended",
      units: "m",
      language: "ja",
      geometry: true,
      instructions: true,
      elevation: false
    };

    console.log("🚗 ルート計算開始:", profile);
    console.log("📍 出発:", [currentLocation.lat, currentLocation.lng]);
    console.log("📍 目的地:", [destination.lat, destination.lng]);

    try {
      const response = await fetch(orsUrl, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json, application/geo+json',
          'Authorization': '5b3ce3597851110001cf6248b8d8f6e4d58e4a60b4c3ec84c0a35c2e'
        },
        body: JSON.stringify(requestBody)
      });

      if (!response.ok) {
        const errorText = await response.text();
        console.error("❌ ORS APIエラー:", response.status, errorText);
        throw new Error('ルート取得失敗');
      }

      const data = await response.json();
      console.log("✅ ルートデータ取得成功");
      
      const feature = data.features[0];
      const geometry = feature.geometry;
      const properties = feature.properties;
      
      // 🔥 重要：GeoJSONの座標は [lng, lat] なので [lat, lng] に変換
      const coordinates = geometry.coordinates.map(coord => [coord[1], coord[0]]);
      
      console.log("📊 座標変換完了:", coordinates.length, "ポイント");
      console.log("🗺️ 最初の3点:", coordinates.slice(0, 3));
      
      setRouteCoordinates(coordinates);

      const distanceKm = (properties.summary.distance / 1000).toFixed(1);
      const durationMin = Math.round(properties.summary.duration / 60);
      
      const now = new Date();
      const arrival = new Date(now.getTime() + properties.summary.duration * 1000);

      // ナビ案内用のステップ
      const steps = properties.segments[0]?.steps || [];
      const navInstructions = steps.map(step => ({
        distance: step.distance,
        duration: step.duration,
        instruction: step.instruction,
        name: step.name || "道なり",
        type: step.type
      }));

      setRouteInfo({
        distance: distanceKm,
        duration: durationMin,
        arrival: arrival.toLocaleTimeString('ja-JP', { hour: '2-digit', minute: '2-digit' }),
        instructions: navInstructions
      });

      console.log("🎯 ルート情報設定完了:", distanceKm, "km,", durationMin, "分");
      console.log("📋 ナビ案内:", navInstructions.length, "ステップ");

      await fetchTrafficInfo(currentLocation, destination);
    } catch (error) {
      console.error("❌ ルート計算エラー:", error);
      throw error;
    }
  };

  const calculateTransitRoute = async () => {
    // 電車ルートの計算（最寄り駅検索→乗り換え案内→目的地駅からの徒歩）
    const prompt = `
    出発地: 緯度${currentLocation.lat}, 経度${currentLocation.lng}
    目的地: ${destination.name} (緯度${destination.lat}, 経度${destination.lng})
    
    以下の情報を返してください:
    1. 出発地から最寄りの駅（名前、緯度、経度、徒歩時間）
    2. 目的地に最も近い駅（名前、緯度、経度）
    3. その間の乗り換え案内（乗車駅、降車駅、路線名、所要時間）
    4. 目的地最寄り駅から目的地までの徒歩時間
    5. 合計所要時間と料金
    `;

    const transitInfo = await base44.integrations.Core.InvokeLLM({
      prompt: prompt,
      add_context_from_internet: true,
      response_json_schema: {
        type: "object",
        properties: {
          nearest_station: {
            type: "object",
            properties: {
              name: { type: "string" },
              lat: { type: "number" },
              lng: { type: "number" },
              walk_time: { type: "number" }
            }
          },
          destination_station: {
            type: "object",
            properties: {
              name: { type: "string" },
              lat: { type: "number" },
              lng: { type: "number" }
            }
          },
          transfers: {
            type: "array",
            items: {
              type: "object",
              properties: {
                line: { type: "string" },
                from: { type: "string" },
                to: { type: "string" },
                duration: { type: "number" }
              }
            }
          },
          destination_walk_time: { type: "number" },
          total_duration: { type: "number" },
          fare: { type: "number" }
        }
      }
    });

    setTransitDetails(transitInfo);

    // 徒歩ルート1: 現在地→最寄り駅
    const walkRoute1 = await calculateWalkingSegment(
      currentLocation.lat,
      currentLocation.lng,
      transitInfo.nearest_station.lat,
      transitInfo.nearest_station.lng
    );

    // 徒歩ルート2: 目的地最寄り駅→目的地
    const walkRoute2 = await calculateWalkingSegment(
      transitInfo.destination_station.lat,
      transitInfo.destination_station.lng,
      destination.lat,
      destination.lng
    );

    // 全ルートを結合（簡易的に直線で駅間を繋ぐ）
    const stationRoute = [
      [transitInfo.nearest_station.lat, transitInfo.nearest_station.lng],
      [transitInfo.destination_station.lat, transitInfo.destination_station.lng]
    ];

    const fullRoute = [...walkRoute1, ...stationRoute, ...walkRoute2];
    setRouteCoordinates(fullRoute);

    const now = new Date();
    const arrival = new Date(now.getTime() + transitInfo.total_duration * 60000);

    setRouteInfo({
      distance: (transitInfo.total_duration * 0.5).toFixed(1),
      duration: transitInfo.total_duration,
      arrival: arrival.toLocaleTimeString('ja-JP', { hour: '2-digit', minute: '2-digit' })
    });
  };

  const calculateWalkingSegment = async (fromLat, fromLng, toLat, toLng) => {
    try {
      const orsUrl = `https://api.openrouteservice.org/v2/directions/foot-walking/geojson`;
      
      const requestBody = {
        coordinates: [
          [fromLng, fromLat],
          [toLng, toLat]
        ]
      };

      const response = await fetch(orsUrl, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/geo+json',
          'Authorization': '5b3ce3597851110001cf6248b8d8f6e4d58e4a60b4c3ec84c0a35c2e'
        },
        body: JSON.stringify(requestBody)
      });

      if (response.ok) {
        const data = await response.json();
        return data.features[0].geometry.coordinates.map(coord => [coord[1], coord[0]]);
      }
    } catch (error) {
      console.error("徒歩ルート取得エラー:", error);
    }

    return [[fromLat, fromLng], [toLat, toLng]];
  };

  const generateSimpleRouteSync = () => {
    // 簡易的なルート座標を即座に生成
    const points = [];
    const steps = 50;
    
    for (let i = 0; i <= steps; i++) {
      const t = i / steps;
      
      // より自然な曲線を生成
      const lat = currentLocation.lat + (destination.lat - currentLocation.lat) * t;
      const lng = currentLocation.lng + (destination.lng - currentLocation.lng) * t;
      
      // 軽微な曲がりを追加
      const offset = Math.sin(t * Math.PI) * 0.005;
      
      points.push([lat + offset, lng + offset]);
    }

    return points;
  };

  const fetchTrafficInfo = async (start, end) => {
    try {
      const prompt = `${start.name}から${end.name}までの現在の交通状況や渋滞情報を簡潔に教えてください。主要道路の混雑状況や所要時間への影響を1-2文で。`;
      
      const response = await base44.integrations.Core.InvokeLLM({
        prompt: prompt,
        add_context_from_internet: true
      });

      setTrafficInfo(response);
    } catch (error) {
      console.error("交通情報取得エラー:", error);
    }
  };

  const toRad = (value) => value * Math.PI / 180;

  const handleTextSend = async () => {
    if (!textInput.trim()) return;
    await handleVoiceCommand(textInput);
    setTextInput("");
  };

  const startNavMode = () => {
    if (destination && currentLocation) {
      setNavMode(true);
      setShowChat(true);
      
      const trafficMsg = trafficInfo ? `\n\n交通情報: ${trafficInfo}` : "";
      const message = `${destination.name}までナビゲーションを開始します！所要時間は約${routeInfo?.duration}分、距離は${routeInfo?.distance}kmです。${trafficMsg}`;
      
      setMessages(prev => [...prev, {
        role: "assistant",
        content: message
      }]);
      
      // 音声で案内
      if ('speechSynthesis' in window) {
        const utterance = new SpeechSynthesisUtterance(
          `${destination.name}までナビゲーションを開始します。所要時間は約${routeInfo?.duration}分です。`
        );
        utterance.lang = 'ja-JP';
        window.speechSynthesis.speak(utterance);
      }
    }
  };

  const transportModes = [
    { id: "driving", icon: Car, label: "車" },
    { id: "walking", icon: Footprints, label: "徒歩" },
    { id: "transit", icon: Train, label: "電車" },
    { id: "cycling", icon: Bike, label: "自転車" }
  ];

  // ナビモードの表示
  if (navMode) {
    return (
      <div className="fixed inset-0 z-50 bg-slate-900">
        {/* 地図 */}
        <NavMap
          currentLocation={currentLocation}
          destination={destination}
          routeCoordinates={routeCoordinates}
          passedCoordinates={passedCoordinates}
          transportMode={transportMode}
          settings={settings}
          transitDetails={transitDetails}
        />

        {/* ナビ案内バー（上部中央） */}
        {currentInstruction && (
          <motion.div
            initial={{ y: -100 }}
            animate={{ y: 0 }}
            className="absolute top-4 left-1/2 -translate-x-1/2 z-[1001] w-full max-w-2xl px-4"
          >
            <Card className="bg-gradient-to-r from-blue-600 to-cyan-600 text-white shadow-2xl border-0">
              <div className="p-4">
                <div className="flex items-center gap-4">
                  {/* 方向アイコン */}
                  <div className="w-16 h-16 bg-white/20 rounded-full flex items-center justify-center flex-shrink-0">
                    <Navigation className="w-8 h-8 text-white" style={{ 
                      transform: currentInstruction.type === 'left' ? 'rotate(-90deg)' : 
                                 currentInstruction.type === 'right' ? 'rotate(90deg)' : 
                                 'rotate(0deg)' 
                    }} />
                  </div>
                  
                  {/* 案内テキスト */}
                  <div className="flex-1">
                    {distanceToNext !== null && (
                      <p className="text-3xl font-bold mb-1">{distanceToNext}m</p>
                    )}
                    <p className="text-lg font-semibold">{currentInstruction.instruction}</p>
                    {currentInstruction.name && currentInstruction.name !== "道なり" && (
                      <p className="text-sm opacity-90 mt-1">{currentInstruction.name}</p>
                    )}
                  </div>

                  {/* 次の案内 */}
                  {nextInstruction && (
                    <div className="text-right flex-shrink-0">
                      <p className="text-xs opacity-75">次</p>
                      <p className="text-sm font-semibold">{Math.round(nextInstruction.distance)}m</p>
                    </div>
                  )}
                </div>
              </div>
            </Card>
          </motion.div>
        )}

        {/* ヘッダー情報 */}
        <div className="absolute top-24 left-0 right-0 z-[1000] p-4">
          <div className="max-w-7xl mx-auto">
            <div className="flex items-center justify-between mb-4">
              <div className="flex items-center gap-3">
                <div className="w-10 h-10 bg-white/90 backdrop-blur rounded-full flex items-center justify-center shadow-lg">
                  <Navigation className="w-5 h-5 text-blue-600" />
                </div>
                <div className="bg-white/90 backdrop-blur px-4 py-2 rounded-full shadow-lg">
                  <h2 className="text-base font-bold text-slate-900">{destination?.name}</h2>
                </div>
              </div>
              <div className="flex gap-2">
                <Button
                  size="icon"
                  variant="ghost"
                  className="bg-white/90 backdrop-blur hover:bg-white shadow-lg rounded-full"
                  onClick={() => setShowSettings(!showSettings)}
                >
                  <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z" />
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />
                  </svg>
                </Button>
                <Button
                  size="icon"
                  variant="ghost"
                  className="bg-white/90 backdrop-blur hover:bg-white shadow-lg rounded-full"
                  onClick={() => setNavMode(false)}
                >
                  <Minimize2 className="w-5 h-5" />
                </Button>
              </div>
            </div>

            {/* ルート情報バー */}
            {routeInfo && (
              <div className="grid grid-cols-3 gap-2">
                <Card className="bg-white/95 backdrop-blur p-2 border-0 shadow-lg">
                  <div className="flex items-center gap-2">
                    <Clock className="w-4 h-4 text-blue-600" />
                    <div>
                      <p className="text-xl font-bold text-blue-600">{routeInfo.duration}</p>
                      <p className="text-xs text-slate-600">分</p>
                    </div>
                  </div>
                </Card>
                <Card className="bg-white/95 backdrop-blur p-2 border-0 shadow-lg">
                  <div className="flex items-center gap-2">
                    <Navigation className="w-4 h-4 text-green-600" />
                    <div>
                      <p className="text-xl font-bold text-green-600">{routeInfo.distance}</p>
                      <p className="text-xs text-slate-600">km</p>
                    </div>
                  </div>
                </Card>
                <Card className="bg-white/95 backdrop-blur p-2 border-0 shadow-lg">
                  <div className="flex items-center gap-2">
                    <Clock className="w-4 h-4 text-purple-600" />
                    <div>
                      <p className="text-lg font-bold text-purple-600">{routeInfo.arrival}</p>
                      <p className="text-xs text-slate-600">到着</p>
                    </div>
                  </div>
                </Card>
              </div>
            )}

            {/* 交通情報 */}
            {trafficInfo && (
              <Card className="bg-yellow-50 border-yellow-200 p-3 mt-3">
                <div className="flex items-start gap-2">
                  <AlertTriangle className="w-5 h-5 text-yellow-600 flex-shrink-0 mt-0.5" />
                  <div>
                    <p className="text-sm font-semibold text-yellow-900">交通情報</p>
                    <p className="text-xs text-yellow-800 mt-1">{trafficInfo}</p>
                  </div>
                </div>
              </Card>
            )}

            {/* 電車ルート詳細 */}
            {transitDetails && (
              <Card className="bg-white/95 backdrop-blur p-4 mt-3">
                <h4 className="font-bold text-sm mb-3 text-slate-900">乗り換え案内</h4>
                <div className="space-y-3 text-xs">
                  <div className="flex items-center gap-2">
                    <Footprints className="w-4 h-4 text-blue-600" />
                    <span>{transitDetails.nearest_station?.name}まで徒歩 {transitDetails.nearest_station?.walk_time}分</span>
                  </div>
                  {transitDetails.transfers?.map((transfer, idx) => (
                    <div key={idx} className="flex items-center gap-2">
                      <Train className="w-4 h-4 text-green-600" />
                      <span>{transfer.line}: {transfer.from} → {transfer.to} ({transfer.duration}分)</span>
                    </div>
                  ))}
                  <div className="flex items-center gap-2">
                    <Footprints className="w-4 h-4 text-blue-600" />
                    <span>徒歩 {transitDetails.destination_walk_time}分</span>
                  </div>
                  <div className="pt-2 border-t">
                    <span className="font-bold">料金: ¥{transitDetails.fare}</span>
                  </div>
                </div>
              </Card>
            )}

            {/* 設定パネル */}
            <AnimatePresence>
              {showSettings && (
                <motion.div
                  initial={{ opacity: 0, y: -20 }}
                  animate={{ opacity: 1, y: 0 }}
                  exit={{ opacity: 0, y: -20 }}
                >
                  <Card className="bg-white/95 backdrop-blur p-4 mt-3">
                    <h4 className="font-bold text-sm mb-3">ナビ設定</h4>
                    <div className="space-y-3">
                      <div>
                        <label className="text-xs text-slate-600">ルート色</label>
                        <Input
                          type="color"
                          value={settings.routeColor}
                          onChange={(e) => setSettings({...settings, routeColor: e.target.value})}
                          className="h-8 mt-1"
                        />
                      </div>
                      <div>
                        <label className="text-xs text-slate-600">通過済み色</label>
                        <Input
                          type="color"
                          value={settings.passedRouteColor}
                          onChange={(e) => setSettings({...settings, passedRouteColor: e.target.value})}
                          className="h-8 mt-1"
                        />
                      </div>
                      <div>
                        <label className="text-xs text-slate-600">線の太さ: {settings.routeWidth}px</label>
                        <Input
                          type="range"
                          min="4"
                          max="16"
                          value={settings.routeWidth}
                          onChange={(e) => setSettings({...settings, routeWidth: parseInt(e.target.value)})}
                          className="mt-1"
                        />
                      </div>
                      <div className="flex items-center gap-2">
                        <input
                          type="checkbox"
                          checked={settings.autoReroute}
                          onChange={(e) => setSettings({...settings, autoReroute: e.target.checked})}
                          className="w-4 h-4"
                        />
                        <label className="text-xs text-slate-600">自動リルート</label>
                      </div>
                    </div>
                  </Card>
                </motion.div>
              )}
            </AnimatePresence>
          </div>
        </div>

        {/* AIチャット（右側） */}
        <AnimatePresence>
          {showChat && (
            <motion.div
              initial={{ x: "100%" }}
              animate={{ x: 0 }}
              exit={{ x: "100%" }}
              className="absolute right-0 top-0 bottom-0 w-full md:w-96 z-[1001]"
            >
              <Card className="h-full bg-white/95 backdrop-blur-xl border-0 flex flex-col shadow-2xl">
                <div className="bg-gradient-to-r from-blue-600 to-cyan-600 text-white p-4 flex items-center justify-between">
                  <div className="flex items-center gap-2">
                    <MessageSquare className="w-5 h-5" />
                    <h3 className="font-bold">AIナビゲーター</h3>
                  </div>
                  <Button
                    size="icon"
                    variant="ghost"
                    className="text-white hover:bg-white/20"
                    onClick={() => setShowChat(false)}
                  >
                    <X className="w-5 h-5" />
                  </Button>
                </div>

                <div className="flex-1 overflow-y-auto p-4 space-y-4">
                  {messages.map((msg, idx) => (
                    <div
                      key={idx}
                      className={`flex ${msg.role === "user" ? "justify-end" : "justify-start"}`}
                    >
                      <div
                        className={`max-w-[80%] p-3 rounded-2xl ${
                          msg.role === "user"
                            ? "bg-blue-600 text-white"
                            : "bg-slate-100 text-slate-900"
                        }`}
                      >
                        {msg.content}
                      </div>
                    </div>
                  ))}
                  {isAIThinking && (
                    <div className="flex justify-start">
                      <div className="bg-slate-100 p-3 rounded-2xl">
                        <Loader2 className="w-5 h-5 animate-spin text-blue-600" />
                      </div>
                    </div>
                  )}
                  <div ref={messagesEndRef} />
                </div>

                <div className="p-4 border-t space-y-3">
                  <div className="flex gap-2">
                    <Input
                      value={textInput}
                      onChange={(e) => setTextInput(e.target.value)}
                      onKeyPress={(e) => e.key === "Enter" && handleTextSend()}
                      placeholder="メッセージを入力..."
                      disabled={isAIThinking}
                    />
                    <Button
                      size="icon"
                      onClick={handleTextSend}
                      disabled={isAIThinking || !textInput.trim()}
                      className="bg-blue-600"
                    >
                      <Send className="w-4 h-4" />
                    </Button>
                  </div>

                  <Button
                    onClick={toggleListening}
                    className={`w-full h-14 text-lg font-bold ${
                      isListening
                        ? "bg-red-500 hover:bg-red-600 animate-pulse"
                        : "bg-gradient-to-r from-blue-600 to-cyan-600"
                    }`}
                  >
                    {isListening ? (
                      <>
                        <MicOff className="w-6 h-6 mr-2" />
                        {transcript || "聞き取り中..."}
                      </>
                    ) : (
                      <>
                        <Mic className="w-6 h-6 mr-2" />
                        タップして話す
                      </>
                    )}
                  </Button>
                </div>
              </Card>
            </motion.div>
          )}
        </AnimatePresence>

        {/* チャット開閉ボタン */}
        {!showChat && (
          <motion.div
            initial={{ scale: 0 }}
            animate={{ scale: 1 }}
            className="absolute right-4 bottom-24 z-[1000]"
          >
            <Button
              size="icon"
              onClick={() => setShowChat(true)}
              className="w-16 h-16 rounded-full bg-gradient-to-r from-blue-600 to-cyan-600 shadow-2xl"
            >
              <MessageSquare className="w-8 h-8 text-white" />
            </Button>
          </motion.div>
        )}

        {/* 音声ボタン（下部中央） */}
        <motion.div
          initial={{ y: 100 }}
          animate={{ y: 0 }}
          className="absolute bottom-8 left-1/2 -translate-x-1/2 z-[1000]"
        >
          <Button
            onClick={toggleListening}
            className={`w-20 h-20 rounded-full shadow-2xl ${
              isListening
                ? "bg-red-500 hover:bg-red-600 animate-pulse"
                : "bg-gradient-to-r from-blue-600 to-cyan-600"
            }`}
          >
            {isListening ? (
              <MicOff className="w-10 h-10 text-white" />
            ) : (
              <Mic className="w-10 h-10 text-white" />
            )}
          </Button>
        </motion.div>
      </div>
    );
  }

  // 通常モードの表示（Googleマップ風レイアウト）
  return (
    <div className="fixed inset-0 bg-slate-100">
      {/* 地図 */}
      <div className="absolute inset-0">
        <NavMap
          currentLocation={currentLocation}
          destination={destination}
          routeCoordinates={routeCoordinates}
          passedCoordinates={passedCoordinates}
          transportMode={transportMode}
          settings={settings}
          transitDetails={transitDetails}
        />
      </div>

      {/* 検索バー（上部） */}
      <div className="absolute top-0 left-0 right-0 z-[1000] p-4 bg-gradient-to-b from-white to-transparent">
        <div className="max-w-2xl mx-auto">
          <form onSubmit={handleSearch}>
            <div className="flex gap-2">
              <div className="flex-1 relative">
                <Search className="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 text-slate-400 z-10" />
                <Input
                  type="text"
                  value={searchQuery}
                  onChange={(e) => setSearchQuery(e.target.value)}
                  placeholder="目的地を検索..."
                  className="pl-12 h-14 text-base bg-white/95 backdrop-blur shadow-lg border-0"
                  disabled={isSearching}
                />
              </div>
              <Button
                type="submit"
                size="lg"
                disabled={isSearching}
                className="h-14 px-6 bg-blue-600 hover:bg-blue-700 shadow-lg"
              >
                {isSearching ? (
                  <Loader2 className="w-5 h-5 animate-spin" />
                ) : (
                  "検索"
                )}
              </Button>
            </div>
          </form>
        </div>
      </div>

      {/* 移動手段選択（上部右） */}
      <div className="absolute top-20 right-4 z-[1000]">
        <Card className="bg-white/95 backdrop-blur shadow-xl p-2">
          <div className="flex flex-col gap-2">
            {transportModes.map(({ id, icon: Icon, label }) => (
              <Button
                key={id}
                variant={transportMode === id ? "default" : "outline"}
                className={`w-12 h-12 p-0 ${
                  transportMode === id ? "bg-blue-600" : "bg-white"
                }`}
                onClick={() => setTransportMode(id)}
                title={label}
              >
                <Icon className="w-6 h-6" />
              </Button>
            ))}
          </div>
        </Card>
      </div>

      {/* 設定ボタン（上部左） */}
      <div className="absolute top-20 left-4 z-[1000] space-y-2">
        <Button
          size="icon"
          onClick={() => setShowSettings(!showSettings)}
          className="w-12 h-12 bg-white/95 backdrop-blur shadow-xl hover:bg-white"
          variant="outline"
        >
          <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z" />
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" />
          </svg>
        </Button>
        {routeInfo && (
          <Button
            size="icon"
            onClick={() => setShowRouteDetails(!showRouteDetails)}
            className="w-12 h-12 bg-white/95 backdrop-blur shadow-xl hover:bg-white"
            variant="outline"
            title="ルート詳細"
          >
            <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2m-3 7h3m-3 4h3m-6-4h.01M9 16h.01" />
            </svg>
          </Button>
        )}
      </div>

      {/* ルート詳細パネル */}
      <AnimatePresence>
        {showRouteDetails && routeInfo && (
          <motion.div
            initial={{ x: -300 }}
            animate={{ x: 0 }}
            exit={{ x: -300 }}
            className="absolute top-56 left-4 z-[1001] w-80"
          >
            <Card className="bg-white/95 backdrop-blur shadow-2xl p-4">
              <div className="flex items-center justify-between mb-4">
                <h3 className="font-bold text-lg">ルート詳細</h3>
                <Button size="icon" variant="ghost" onClick={() => setShowRouteDetails(false)}>
                  <X className="w-4 h-4" />
                </Button>
              </div>
              <div className="space-y-3 text-sm">
                <div className="flex justify-between items-center p-2 bg-blue-50 rounded">
                  <span className="text-slate-600">総距離</span>
                  <span className="font-bold text-blue-600">{routeInfo.distance} km</span>
                </div>
                <div className="flex justify-between items-center p-2 bg-green-50 rounded">
                  <span className="text-slate-600">所要時間</span>
                  <span className="font-bold text-green-600">{routeInfo.duration} 分</span>
                </div>
                <div className="flex justify-between items-center p-2 bg-purple-50 rounded">
                  <span className="text-slate-600">到着予定</span>
                  <span className="font-bold text-purple-600">{routeInfo.arrival}</span>
                </div>
                {routeInfo.roadNames && routeInfo.roadNames.length > 0 && (
                  <div className="p-2 bg-slate-50 rounded">
                    <p className="font-semibold text-slate-700 mb-2">主要道路</p>
                    <ul className="space-y-1">
                      {routeInfo.roadNames.slice(0, 5).map((road, idx) => (
                        <li key={idx} className="text-slate-600 text-xs">• {road}</li>
                      ))}
                    </ul>
                  </div>
                )}
                <div className="p-2 bg-slate-50 rounded">
                  <p className="font-semibold text-slate-700 mb-1">移動手段</p>
                  <p className="text-slate-600 text-xs">
                    {transportMode === "driving" ? "🚗 車" : 
                     transportMode === "walking" ? "🚶 徒歩" : 
                     transportMode === "cycling" ? "🚴 自転車" : "🚃 電車"}
                  </p>
                </div>
              </div>
            </Card>
          </motion.div>
        )}
      </AnimatePresence>

      {/* 設定パネル */}
      <AnimatePresence>
        {showSettings && (
          <motion.div
            initial={{ x: -300 }}
            animate={{ x: 0 }}
            exit={{ x: -300 }}
            className="absolute top-32 left-4 z-[1001] w-80"
          >
            <Card className="bg-white/95 backdrop-blur shadow-2xl p-4">
              <div className="flex items-center justify-between mb-4">
                <h3 className="font-bold text-lg">ナビ設定</h3>
                <Button size="icon" variant="ghost" onClick={() => setShowSettings(false)}>
                  <X className="w-4 h-4" />
                </Button>
              </div>
              <div className="space-y-4">
                <div>
                  <label className="text-sm text-slate-600 font-medium">ルート色</label>
                  <div className="flex items-center gap-3 mt-2">
                    <Input
                      type="color"
                      value={settings.routeColor}
                      onChange={(e) => setSettings({...settings, routeColor: e.target.value})}
                      className="h-10 w-20"
                    />
                    <span className="text-sm text-slate-500">{settings.routeColor}</span>
                  </div>
                </div>
                <div>
                  <label className="text-sm text-slate-600 font-medium">通過済みルート色</label>
                  <div className="flex items-center gap-3 mt-2">
                    <Input
                      type="color"
                      value={settings.passedRouteColor}
                      onChange={(e) => setSettings({...settings, passedRouteColor: e.target.value})}
                      className="h-10 w-20"
                    />
                    <span className="text-sm text-slate-500">{settings.passedRouteColor}</span>
                  </div>
                </div>
                <div>
                  <label className="text-sm text-slate-600 font-medium">線の太さ: {settings.routeWidth}px</label>
                  <Input
                    type="range"
                    min="4"
                    max="16"
                    value={settings.routeWidth}
                    onChange={(e) => setSettings({...settings, routeWidth: parseInt(e.target.value)})}
                    className="mt-2"
                  />
                </div>
                <div>
                  <label className="text-sm text-slate-600 font-medium">リルート距離: {settings.rerouteThreshold}m</label>
                  <Input
                    type="range"
                    min="20"
                    max="200"
                    step="10"
                    value={settings.rerouteThreshold}
                    onChange={(e) => setSettings({...settings, rerouteThreshold: parseInt(e.target.value)})}
                    className="mt-2"
                  />
                </div>
                <div className="flex items-center gap-3 p-3 bg-slate-50 rounded-lg">
                  <input
                    type="checkbox"
                    checked={settings.autoReroute}
                    onChange={(e) => setSettings({...settings, autoReroute: e.target.checked})}
                    className="w-5 h-5"
                    id="autoReroute"
                  />
                  <label htmlFor="autoReroute" className="text-sm text-slate-600 font-medium cursor-pointer">
                    自動リルート
                  </label>
                </div>
              </div>
            </Card>
          </motion.div>
        )}
      </AnimatePresence>

      {/* ルート情報パネル（下部スライドアップ） */}
      <AnimatePresence>
        {destination && (
          <motion.div
            initial={{ y: "100%" }}
            animate={{ y: 0 }}
            exit={{ y: "100%" }}
            className="absolute bottom-0 left-0 right-0 z-[1000]"
          >
            <Card className="bg-white/95 backdrop-blur-xl shadow-2xl border-0 rounded-t-3xl">
              <div className="p-6 space-y-4">
                {/* 目的地情報 */}
                <div className="flex items-start justify-between">
                  <div className="flex-1">
                    <h2 className="text-2xl font-bold text-slate-900">{destination.name}</h2>
                    {destination.address && (
                      <p className="text-sm text-slate-500 mt-1">{destination.address}</p>
                    )}
                  </div>
                  <Button
                    size="icon"
                    variant="ghost"
                    onClick={() => {
                      setDestination(null);
                      setRouteInfo(null);
                      setRouteCoordinates(null);
                      setTrafficInfo(null);
                    }}
                  >
                    <X className="w-5 h-5" />
                  </Button>
                </div>

                {/* ルート計算中 */}
                {isCalculatingRoute && (
                  <div className="flex items-center justify-center py-4">
                    <Loader2 className="w-6 h-6 animate-spin text-blue-600 mr-2" />
                    <span className="text-slate-600">ルートを計算中...</span>
                  </div>
                )}

                {/* ルート情報 */}
                {routeInfo && (
                  <>
                    <div className="grid grid-cols-3 gap-3">
                      <div className="text-center p-3 bg-blue-50 rounded-xl">
                        <Clock className="w-5 h-5 text-blue-600 mx-auto mb-1" />
                        <p className="text-2xl font-bold text-blue-600">{routeInfo.duration}</p>
                        <p className="text-xs text-slate-600">分</p>
                      </div>
                      <div className="text-center p-3 bg-green-50 rounded-xl">
                        <Navigation className="w-5 h-5 text-green-600 mx-auto mb-1" />
                        <p className="text-2xl font-bold text-green-600">{routeInfo.distance}</p>
                        <p className="text-xs text-slate-600">km</p>
                      </div>
                      <div className="text-center p-3 bg-purple-50 rounded-xl">
                        <Clock className="w-5 h-5 text-purple-600 mx-auto mb-1" />
                        <p className="text-xl font-bold text-purple-600">{routeInfo.arrival}</p>
                        <p className="text-xs text-slate-600">到着</p>
                      </div>
                    </div>

                    {/* 交通情報 */}
                    {trafficInfo && (
                      <div className="bg-yellow-50 border border-yellow-200 rounded-lg p-3">
                        <div className="flex items-start gap-2">
                          <AlertTriangle className="w-5 h-5 text-yellow-600 flex-shrink-0 mt-0.5" />
                          <div>
                            <p className="text-sm font-semibold text-yellow-900">交通情報</p>
                            <p className="text-xs text-yellow-800 mt-1">{trafficInfo}</p>
                          </div>
                        </div>
                      </div>
                    )}

                    {/* 電車ルート詳細 */}
                    {transitDetails && (
                      <div className="bg-slate-50 rounded-lg p-3">
                        <h4 className="font-bold text-sm mb-2 text-slate-900">乗り換え案内</h4>
                        <div className="space-y-2 text-xs">
                          <div className="flex items-center gap-2">
                            <Footprints className="w-4 h-4 text-blue-600" />
                            <span>{transitDetails.nearest_station?.name}まで徒歩 {transitDetails.nearest_station?.walk_time}分</span>
                          </div>
                          {transitDetails.transfers?.map((transfer, idx) => (
                            <div key={idx} className="flex items-center gap-2">
                              <Train className="w-4 h-4 text-green-600" />
                              <span>{transfer.line}: {transfer.from} → {transfer.to} ({transfer.duration}分)</span>
                            </div>
                          ))}
                          <div className="flex items-center gap-2">
                            <Footprints className="w-4 h-4 text-blue-600" />
                            <span>徒歩 {transitDetails.destination_walk_time}分</span>
                          </div>
                          <div className="pt-2 border-t">
                            <span className="font-bold">料金: ¥{transitDetails.fare}</span>
                          </div>
                        </div>
                      </div>
                    )}

                    {/* ナビ開始ボタン */}
                    <Button
                      onClick={startNavMode}
                      disabled={!routeCoordinates || isCalculatingRoute}
                      className="w-full h-16 text-xl font-bold bg-gradient-to-r from-blue-600 to-cyan-600 hover:from-blue-700 hover:to-cyan-700 text-white shadow-xl"
                    >
                      <Navigation className="w-7 h-7 mr-2" />
                      ナビを開始
                    </Button>
                  </>
                )}
              </div>
            </Card>
          </motion.div>
        )}
      </AnimatePresence>

      {/* 音声ボタン（右下） */}
      <motion.div
        initial={{ scale: 0 }}
        animate={{ scale: 1 }}
        className="absolute right-4 bottom-6 z-[999]"
      >
        <Button
          onClick={toggleListening}
          className={`w-16 h-16 rounded-full shadow-2xl ${
            isListening
              ? "bg-red-500 hover:bg-red-600 animate-pulse"
              : "bg-gradient-to-r from-blue-600 to-cyan-600"
          }`}
        >
          {isListening ? (
            <MicOff className="w-8 h-8 text-white" />
          ) : (
            <Mic className="w-8 h-8 text-white" />
          )}
        </Button>
        {transcript && isListening && (
          <motion.div
            initial={{ opacity: 0, y: 10 }}
            animate={{ opacity: 1, y: 0 }}
            className="absolute bottom-20 right-0 bg-white/95 backdrop-blur px-4 py-2 rounded-lg shadow-xl whitespace-nowrap"
          >
            <p className="text-sm text-slate-700">{transcript}</p>
          </motion.div>
        )}
      </motion.div>

      {/* AIチャットボタン */}
      <motion.div
        initial={{ scale: 0 }}
        animate={{ scale: 1 }}
        className="absolute right-4 bottom-28 z-[999]"
      >
        <Button
          onClick={() => setShowChat(!showChat)}
          className="w-14 h-14 rounded-full bg-white shadow-xl hover:bg-slate-50"
          variant="outline"
        >
          <MessageSquare className="w-6 h-6 text-blue-600" />
        </Button>
      </motion.div>

      {/* AIチャットパネル */}
      <AnimatePresence>
        {showChat && (
          <motion.div
            initial={{ x: "100%" }}
            animate={{ x: 0 }}
            exit={{ x: "100%" }}
            className="absolute right-0 top-0 bottom-0 w-full md:w-96 z-[1001]"
          >
            <Card className="h-full bg-white/95 backdrop-blur-xl shadow-2xl border-0 flex flex-col">
              <div className="bg-gradient-to-r from-blue-600 to-cyan-600 text-white p-4 flex items-center justify-between">
                <div className="flex items-center gap-2">
                  <MessageSquare className="w-5 h-5" />
                  <h3 className="font-bold">AIアシスタント</h3>
                </div>
                <Button size="icon" variant="ghost" className="text-white hover:bg-white/20" onClick={() => setShowChat(false)}>
                  <X className="w-5 h-5" />
                </Button>
              </div>

              <div className="flex-1 overflow-y-auto p-4 space-y-4">
                {messages.map((msg, idx) => (
                  <div key={idx} className={`flex ${msg.role === "user" ? "justify-end" : "justify-start"}`}>
                    <div className={`max-w-[80%] p-3 rounded-2xl ${
                      msg.role === "user" ? "bg-blue-600 text-white" : "bg-slate-100 text-slate-900"
                    }`}>
                      {msg.content}
                    </div>
                  </div>
                ))}
                {isAIThinking && (
                  <div className="flex justify-start">
                    <div className="bg-slate-100 p-3 rounded-2xl">
                      <Loader2 className="w-5 h-5 animate-spin text-blue-600" />
                    </div>
                  </div>
                )}
                <div ref={messagesEndRef} />
              </div>

              <div className="p-4 border-t space-y-3">
                <div className="flex gap-2">
                  <Input
                    value={textInput}
                    onChange={(e) => setTextInput(e.target.value)}
                    onKeyPress={(e) => e.key === "Enter" && handleTextSend()}
                    placeholder="メッセージを入力..."
                    disabled={isAIThinking}
                  />
                  <Button size="icon" onClick={handleTextSend} disabled={isAIThinking || !textInput.trim()} className="bg-blue-600">
                    <Send className="w-4 h-4" />
                  </Button>
                </div>
              </div>
            </Card>
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
}
