# ktotalcare
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>K-TOTAL CARE 플랫폼</title>
    <!-- Tailwind CSS (Tailwind JIT CDN) -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Font Awesome Icons -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css">
    <!-- Google Fonts: Inter -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap" rel="stylesheet">
    <style>
        /* Custom Styles for Aesthetics and Responsiveness */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f4f6f8;
            color: #1f2937;
        }
        .container-fluid {
            width: 100%;
            max-width: 1200px;
            margin: 0 auto;
            padding: 0 1rem;
        }
        .header-bg {
            background-color: #0b2447; /* Deep Blue - K-Medical Professionalism */
        }
        .cta-button {
            transition: all 0.2s;
            background-color: #ff6b6b; /* Bright Coral - Attention-grabbing CTA */
        }
        .cta-button:hover {
            background-color: #ff4757;
            transform: translateY(-2px);
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
        .card {
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
            transition: transform 0.3s;
        }
        .card:hover {
            transform: translateY(-5px);
        }
        /* Custom Scrollbar for better UX */
        ::-webkit-scrollbar {
            width: 8px;
        }
        ::-webkit-scrollbar-track {
            background: #f1f1f1;
        }
        ::-webkit-scrollbar-thumb {
            background: #888;
            border-radius: 4px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #555;
        }
        /* Style for the partnership section specific styling */
        .partner-list {
            list-style: none;
            padding: 0;
            margin: 0;
        }
        .partner-list li {
            padding: 0.5rem 0;
            border-bottom: 1px dashed #e5e7eb;
        }
        .partner-list li:last-child {
            border-bottom: none;
        }
    </style>
</head>
<body class="min-h-screen">

    <!-- Firebase SDK Imports (MUST be placed before your script) -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, addDoc, onSnapshot, collection, query, serverTimestamp, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global Variables
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'ktotalcare-default-app-id';
        let firebaseConfig = null;
        try {
            firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        } catch (e) {
            console.error("Firebase config parsing error:", e);
        }

        let app, db, auth, userId = null;
        let isListenerSetup = false; // 리스너 중복 방지를 위한 플래그

        // Set Firebase Debug Log Level (Optional but helpful)
        setLogLevel('Debug'); // 디버그 로깅 활성화

        /**
         * Initialize Firebase and Authenticate User
         */
        async function initializeFirebaseAndAuth() {
            if (!firebaseConfig || Object.keys(firebaseConfig).length === 0) {
                console.error("Firebase configuration is missing or invalid. Data storage will not work.");
                return;
            }

            try {
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);

                // Authentication Listener: 리스너는 인증 완료된 user 객체가 있을 때만 데이터를 로드하도록 엄격하게 처리합니다.
                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        // 인증된 사용자가 있을 때만 userId 설정
                        userId = user.uid;
                        document.getElementById('user-id-display').textContent = userId;
                        console.log("Authenticated with UID:", userId);
                        
                        // 인증 완료 후, 채팅 리스너 설정
                        setupChatListener();
                    } else {
                        // 미인증 상태 또는 초기 로딩 중. Firestore 접근 시도하지 않음.
                        userId = null;
                        console.log("Authentication state change: User is not signed in or authentication is in progress.");
                        
                        // 언어 스크립트가 로드되었는지 확인하고 로딩 메시지 표시
                        let loadingMessage = "인증 중입니다. 잠시만 기다려주세요.";
                        if(typeof window.getL10nText === 'function') {
                            loadingMessage = window.getL10nText('loading_auth_message');
                        }
                        document.getElementById('chat-messages').innerHTML = `<div class="p-4 text-center text-gray-500">${loadingMessage}</div>`;
                    }
                });

                // Sign in using custom token or anonymously
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    // Fallback to anonymous sign-in if token is unavailable
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Firebase Initialization or Authentication failed:", error);
                document.getElementById('chat-messages').innerHTML = `<div class="p-4 text-center text-red-500">데이터베이스 연결 또는 인증 오류: ${error.message}</div>`;
            }
        }

        /**
         * Get the Firestore collection path for the 1:1 chat
         */
        function getChatCollectionPath(currentUserId) {
            // Private data storage path: /artifacts/{appId}/users/{userId}/{collectionName}
            return `artifacts/${appId}/users/${currentUserId}/consultations`;
        }

        /**
         * Setup real-time listener for chat messages
         */
        function setupChatListener() {
            // 리스너가 이미 설정되었거나 인증/DB 준비가 안된 경우 실행하지 않음 (userId가 null이면 미인증 상태)
            if (isListenerSetup || !userId || !db) return; 

            isListenerSetup = true; // 리스너 설정 플래그 ON

            const chatRef = collection(db, getChatCollectionPath(userId));
            const messagesList = document.getElementById('chat-messages');

            // Listen for messages (ordered by timestamp, client-side sorting is preferred for simplicity)
            onSnapshot(chatRef, (snapshot) => {
                let messages = [];
                snapshot.forEach(doc => {
                    messages.push({ id: doc.id, ...doc.data() });
                });

                // Sort messages by timestamp to ensure correct order
                // Firestore timestamp 객체가 없을 수 있으므로 안전하게 처리
                messages.sort((a, b) => (a.timestamp?.toMillis() || 0) - (b.timestamp?.toMillis() || 0));

                messagesList.innerHTML = ''; // Clear existing messages

                if (messages.length === 0) {
                    // 번역된 빈 메시지 텍스트를 가져와서 사용
                    const emptyMessageText = typeof window.getL10nText === 'function' 
                        ? window.getL10nText('chat_empty_message') 
                        : "상담을 시작하려면 아래에 메시지를 입력하세요."; // Fallback
                        
                    messagesList.innerHTML = `<div class="p-4 text-center text-gray-400">${emptyMessageText}</div>`;
                }

                messages.forEach(msg => {
                    const messageElement = document.createElement('div');
                    const isUser = msg.role === 'user';
                    const time = msg.timestamp ? new Date(msg.timestamp.seconds * 1000).toLocaleTimeString('ko-KR', { hour: '2-digit', minute: '2-digit' }) : '방금';

                    messageElement.className = isUser ? 'flex justify-end mb-4' : 'flex justify-start mb-4';
                    
                    messageElement.innerHTML = `
                        <div class="max-w-xs md:max-w-md lg:max-w-lg ${isUser ? 'bg-indigo-500 text-white rounded-l-xl rounded-t-xl' : 'bg-white text-gray-800 rounded-r-xl rounded-t-xl shadow-md'} p-3 break-words">
                            <p class="text-sm">${msg.text}</p>
                            <span class="text-xs mt-1 block ${isUser ? 'text-indigo-200' : 'text-gray-500'} text-right">${time}</span>
                        </div>
                    `;
                    messagesList.appendChild(messageElement);
                });

                // Scroll to the latest message
                messagesList.scrollTop = messagesList.scrollHeight;
            }, (error) => {
                console.error("Error listening to chat data:", error);
                // 권한 오류 발생 시, 사용자에게 명확히 안내
                messagesList.innerHTML = `<div class="p-4 text-center text-red-500">실시간 데이터 로딩 오류: 데이터베이스 권한이 부족합니다. (Error: ${error.message})</div>`;
                isListenerSetup = false; // 오류 발생 시 리스너 재설정을 위해 플래그 해제
            });
        }

        /**
         * Send a new chat message
         */
        window.sendMessage = async function() {
            const input = document.getElementById('chat-input');
            const text = input.value.trim();

            if (!text) return;
            // userId가 설정되어 있고 (인증 완료), db가 준비되었는지 확인
            if (!userId || !db) {
                alert('데이터베이스 연결 및 인증 중입니다. 잠시 후 다시 시도해주세요.');
                return;
            }

            try {
                await addDoc(collection(db, getChatCollectionPath(userId)), {
                    role: 'user',
                    text: text,
                    timestamp: serverTimestamp()
                });
                input.value = ''; // Clear input field
            } catch (error) {
                console.error("Error adding document:", error);
                alert('메시지 전송에 실패했습니다: ' + error.message);
            }
        };

        // Initialize Firebase on window load
        initializeFirebaseAndAuth();

    </script>

    <!-- Google Translate Script -->
    <script type="text/javascript">
        function googleTranslateElementInit() {
            new google.translate.TranslateElement({
                pageLanguage: 'ko',
                layout: google.translate.TranslateElement.InlineLayout.SIMPLE,
                autoDisplay: true
            }, 'google_translate_element');
        }
    </script>
    <script type="text/javascript" src="//translate.google.com/translate_a/element.js?cb=googleTranslateElementInit"></script>

    <!-- Language Switcher and Localization Script -->
    <script>
        // 수정된 주소는 이곳에 하드코딩됩니다.
        const PLATFORM_ADDRESS = "https://chespro-director.github.io/ktotalcare/";
        
        // Define all supported languages for easy iteration
        const SUPPORTED_LANGS = ['ko', 'en', 'zh', 'id', 'th', 'vi', 'ja'];

        const translations = {
            'ko': {
                platform_name: "K-TOTAL CARE 플랫폼",
                subtitle: "K-의료와 토탈 케어 서비스를 한 곳에서!",
                menu_home: "홈",
                menu_about: "플랫폼 소개",
                menu_services: "주요 서비스",
                menu_consultation: "1:1 상담",
                main_title: "대한민국 의료 관광의 새로운 시작",
                main_description: "ktotalcare.com은 인도네시아, 베트남 등 아시아 환자들을 한국의 우수한 의료기관과 연결하는 올인원(All-in-One) 토탈 케어 플랫폼입니다.",
                section_title_1: "왜 K-TOTAL CARE인가?",
                service_1_title: "검증된 병원 네트워크",
                service_1_desc: "의료해외진출법에 등록된 전문 의료기관만 제휴하여 안전하고 신뢰할 수 있는 서비스를 제공합니다.",
                service_2_title: "진료 외 토탈 지원",
                service_2_desc: "교통, 숙박, 전문 통역, K-뷰티 사후 관리 제품 연계까지 모든 과정을 지원합니다.",
                service_3_title: "맞춤형 의료 패키지",
                service_3_desc: "성형, 피부, 건강검진 등 고객의 니즈에 맞춘 최적의 특화 상품 패키지를 제공합니다.",
                consult_title: "1:1 실시간 전문 상담 (Secure Chat)",
                consult_desc: "궁금한 점을 문의하세요. 전문 코디네이터가 신속하고 정확하게 답변해 드립니다.",
                send_button: "전송",
                placeholder_input: "메시지를 입력하세요...",
                user_id_label: "현재 사용자 ID:",
                language_switcher: "언어 선택",
                cta_consult_button: "1:1 상담 신청하기",
                chat_empty_message: "상담을 시작하려면 아래에 메시지를 입력하세요.",
                loading_auth_message: "인증 중입니다. 잠시만 기다려주세요.",
                footer_copyright: "&copy; 2024 K-TOTAL CARE Platform. All rights reserved. | Powered by TAELYN MED, LCS Partners, UNICORE Indonesia.",
                footer_address_prefix: "플랫폼 주소:",
                // 태린메드 사업 개요 (NEW)
                biz_overview_title: "㈜태린메드 사업 개요 및 비전",
                biz_overview_p1: "㈜태린메드는 외국인 환자 유치 및 관리를 위해 설립된 전문 법인입니다. 우리는 엘씨에스파트너스, 유니코아 인도네시아 법인과의 공동 사업을 통해 아시아 전역의 환자들에게 국내의 우수한 의료 서비스를 연결하는 교두보 역할을 수행하고 있습니다. IT 플랫폼 기반의 체계적인 토탈 케어 시스템을 통해 의료 관광 시장을 선도하는 것이 우리의 비전입니다.",
                market_title: "주요 타겟 시장",
                market_1: "인도네시아: 현지 법인(PT UNICORE BIZPAY INDONESIA)을 통한 시장 집중 및 거점 확보",
                market_2: "베트남/태국: 현지 에이전시 및 여행사와의 제휴를 통한 사업 확장",
                market_3: "중국: 대형 여행사와의 전략적 제휴를 통한 시장 진출 및 안정화",
                market_4: "미국: 장기적으로 사업 영역을 확장하여 글로벌 시장 진출",
                service_main_title: "주요 서비스",
                service_main: "IT 플랫폼 기반의 실시간 상담, 병원 예약 및 스케줄 관리, 통역/숙소/교통 연계 서비스 등 환자의 전 여정을 책임지는 '토탈 케어 서비스'를 제공합니다.",
                // 태린메드 고객 신뢰 섹션 (기존)
                trust_title: "주식회사 태린메드 (TAELYN MED) 고객 신뢰 상세 안내",
                trust_desc: "K-TOTAL CARE 플랫폼 운영사인 (주)태린메드는 외국인환자유치 등록 의료기관 및 전문 기관과의 공식 제휴를 통해 고객의 안전과 신뢰를 최우선으로 합니다. 아래 제휴 현황을 확인하실 수 있습니다.",
                trust_hospital_title: "공식 제휴 병원 및 전문기관",
                trust_hospital_list: [
                    "도자기의원 (피부, 성형)",
                    "원데이치과 (치과 진료)",
                    "하늘안과 (안과 진료)",
                    "A 병원 (종합 검진)",
                    "B 성형외과 (성형 수술)",
                    "C 피부과 (피부/미용)",
                    "D 치과 (치아 교정)",
                    "E 한의원 (한방 진료)",
                    "F 건강검진센터 (종합 건강검진)"
                ],
                trust_license_title: "주요 등록 및 인증 현황",
                trust_license_1: "외국인환자유치업 정식 등록 (보건복지부)",
                trust_license_2: "IT 플랫폼 기반 의료 관광 토탈 케어 서비스 제공",
                trust_license_3: "해외 현지 법인 (PT UNICORE BIZPAY INDONESIA)과의 공식 파트너십",
            },
            'en': {
                platform_name: "K-TOTAL CARE Platform",
                subtitle: "K-Medical and Total Care Services in One Place!",
                menu_home: "Home",
                menu_about: "Platform Info",
                menu_services: "Key Services",
                menu_consultation: "1:1 Consultation",
                main_title: "The New Era of Medical Tourism in Korea",
                main_description: "ktotalcare.com is an All-in-One Total Care platform connecting patients from Asia (Indonesia, Vietnam, etc.) with Korea's best medical institutions.",
                section_title_1: "Why K-TOTAL CARE?",
                service_1_title: "Verified Hospital Network",
                service_1_desc: "We partner only with specialized medical institutions registered under the Korean Medical Overseas Expansion Act, ensuring safe and reliable service.",
                service_2_title: "Total Support Beyond Treatment",
                service_2_desc: "We assist with the entire process: transportation, accommodation, professional interpretation, and post-care linkage to K-Beauty products.",
                service_3_title: "Customized Medical Packages",
                service_3_desc: "We offer optimized specialty packages tailored to customer needs, including plastic surgery, dermatology, and comprehensive health check-ups.",
                consult_title: "1:1 Real-Time Professional Consultation (Secure Chat)",
                consult_desc: "Ask any questions you have. Our expert coordinator will provide prompt and accurate answers.",
                send_button: "Send",
                placeholder_input: "Type your message...",
                user_id_label: "Current User ID:",
                language_switcher: "Language Select",
                cta_consult_button: "Apply for 1:1 Consultation",
                chat_empty_message: "Type a message below to start a consultation.",
                loading_auth_message: "Authenticating. Please wait a moment.",
                footer_copyright: "&copy; 2024 K-TOTAL CARE Platform. All rights reserved. | Powered by TAELYN MED, LCS Partners, UNICORE Indonesia.",
                footer_address_prefix: "Platform Address:",
                // 태린메드 사업 개요 (NEW)
                biz_overview_title: "TAELYN MED Business Overview & Vision",
                biz_overview_p1: "TAELYN MED Co., Ltd. is a professional corporation established for attracting and managing foreign patients. Through a joint venture with LCS Partners and UNICORE Indonesia, we serve as a bridge connecting patients across Asia to Korea's excellent medical services. Our vision is to lead the medical tourism market through a systematic, IT platform-based total care system.",
                market_title: "Key Target Markets",
                market_1: "Indonesia: Market focus and establishment of a foothold through the local subsidiary (PT UNICORE BIZPAY INDONESIA)",
                market_2: "Vietnam/Thailand: Business expansion through partnerships with local agencies and travel companies",
                market_3: "China: Market entry and stabilization through strategic alliance with major travel agencies",
                market_4: "USA: Long-term business expansion into the global market",
                service_main_title: "Key Services",
                service_main: "We provide 'Total Care Service' covering the patient's entire journey, including IT platform-based real-time consultation, hospital booking and schedule management, and linkage services for interpretation, accommodation, and transportation.",
                // 태린메드 고객 신뢰 섹션 (기존)
                trust_title: "TAELYN MED Customer Trust Detailed Guide",
                trust_desc: "TAELYN MED Co., Ltd., the operator of the K-TOTAL CARE platform, prioritizes customer safety and trust through official partnerships with medical institutions and specialized agencies registered for attracting foreign patients. You can check the partnership status below.",
                trust_hospital_title: "Official Partner Hospitals and Specialized Institutions",
                trust_hospital_list: [
                    "Dojaki Clinic (Dermatology, Plastic Surgery)",
                    "One Day Dental Clinic (Dental Treatment)",
                    "Haneul Eye Clinic (Ophthalmology)",
                    "A Hospital (General Check-up)",
                    "B Plastic Surgery (Plastic Surgery)",
                    "C Dermatology (Skin/Aesthetics)",
                    "D Dental Clinic (Orthodontics)",
                    "E Oriental Clinic (Traditional Korean Medicine)",
                    "F Health Check-up Center (Comprehensive Health Check-up)"
                ],
                trust_license_title: "Key Registrations and Certifications",
                trust_license_1: "Official Registration as Foreign Patient Attraction Business (Ministry of Health and Welfare)",
                trust_license_2: "Provision of IT Platform-based Medical Tourism Total Care Service",
                trust_license_3: "Official Partnership with Overseas Local Subsidiary (PT UNICORE BIZPAY INDONESIA)",
            },
            'zh': {
                platform_name: "K-TOTAL CARE 平台",
                subtitle: "K-医疗和全方位护理服务一站式体验！",
                menu_home: "首页",
                menu_about: "平台介绍",
                menu_services: "主要服务",
                menu_consultation: "1对1咨询",
                main_title: "韩国医疗旅游的新起点",
                main_description: "ktotalcare.com 是一个一体化的全方位护理平台，将印度尼西亚、越南等亚洲患者与韩国优秀的医疗机构连接起来。",
                section_title_1: "为什么选择 K-TOTAL CARE?",
                service_1_title: "认证医院网络",
                service_1_desc: "我们只与在韩国海外医疗拓展法下注册的专业医疗机构合作，确保提供安全可靠的服务。",
                service_2_title: "诊疗外全面支持",
                service_2_desc: "我们协助全程：交通、住宿、专业翻译，以及与 K-美容善后管理产品的连接。",
                service_3_title: "定制医疗套餐",
                service_3_desc: "提供根据客户需求定制的优化特色套餐，包括整形、皮肤和综合健康检查。",
                consult_title: "1对1 实时专业咨询 (安全聊天)",
                consult_desc: "请咨询您的问题。我们的专业协调员将提供及时准确的答复。",
                send_button: "发送",
                placeholder_input: "输入您的信息...",
                user_id_label: "当前用户 ID:",
                language_switcher: "选择语言",
                cta_consult_button: "申请1对1咨询",
                chat_empty_message: "在下方输入信息开始咨询。",
                loading_auth_message: "正在认证。请稍候。",
                footer_copyright: "&copy; 2024 K-TOTAL CARE 平台. 版权所有. | 由 TAELYN MED, LCS Partners, UNICORE Indonesia 提供支持.",
                footer_address_prefix: "平台地址:",
                // 태린메드 사업 개요 (NEW)
                biz_overview_title: "泰琳医疗(TAELYN MED) 业务概况与愿景",
                biz_overview_p1: "泰琳医疗(TAELYN MED) 是一家为招募和管理外国患者而设立的专业法人。我们通过与LCS Partners和UNICORE印尼法人的合作，为亚洲各地的患者提供连接韩国优质医疗服务的桥梁。我们的愿景是通过基于IT平台的系统化全方位护理系统，引领医疗旅游市场。",
                market_title: "主要目标市场",
                market_1: "印度尼西亚: 通过当地法人(PT UNICORE BIZPAY INDONESIA)实现市场集中和基地建设",
                market_2: "越南/泰国: 通过与当地中介和旅行社的合作进行业务拓展",
                market_3: "中国: 通过与大型旅行社的战略合作进入市场并实现稳定化",
                market_4: "美国: 长期扩大业务范围以进入全球市场",
                service_main_title: "主要服务",
                service_main: "我们提供覆盖患者全程的“全方位护理服务”，包括基于IT平台的实时咨询、医院预约和日程管理，以及口译/住宿/交通关联服务等。",
                // 태린메드 고객 신뢰 섹션 (기존)
                trust_title: "泰琳医疗(TAELYN MED) 客户信任详细指南",
                trust_desc: "K-TOTAL CARE 平台运营商泰琳医疗(TAELYN MED) 通过与注册招募外国患者的医疗机构和专业机构的官方合作，将客户的安全和信任放在首位。您可以在下方查看合作状态。",
                trust_hospital_title: "官方合作医院和专业机构",
                trust_hospital_list: [
                    "Dojaki 诊所 (皮肤, 整形)",
                    "One Day 牙科 (牙科诊疗)",
                    "Haneul 眼科 (眼科诊疗)",
                    "A 医院 (综合体检)",
                    "B 整形外科 (整形手术)",
                    "C 皮肤科 (皮肤/美容)",
                    "D 牙科 (牙齿矫正)",
                    "E 韩国传统医学诊所 (韩方诊疗)",
                    "F 健康体检中心 (综合健康体检)"
                ],
                trust_license_title: "主要注册和认证状态",
                trust_license_1: "正式注册为外国患者招募企业 (韩国保健福祉部)",
                trust_license_2: "提供基于IT平台的医疗旅游全方位护理服务",
                trust_license_3: "与海外当地法人 (PT UNICORE BIZPAY INDONESIA) 的官方合作关系",
            },
            'id': {
                platform_name: "Platform K-TOTAL CARE",
                subtitle: "Layanan K-Medis dan Perawatan Total di Satu Tempat!",
                menu_home: "Beranda",
                menu_about: "Tentang Platform",
                menu_services: "Layanan Utama",
                menu_consultation: "Konsultasi 1:1",
                main_title: "Era Baru Pariwisata Medis di Korea Selatan",
                main_description: "ktotalcare.com adalah platform Perawatan Total All-in-One yang menghubungkan pasien dari Asia (Indonesia, Vietnam, dll.) dengan institusi medis terbaik Korea.",
                section_title_1: "Mengapa K-TOTAL CARE?",
                service_1_title: "Jaringan Rumah Sakit Terverifikasi",
                service_1_desc: "Kami hanya bermitra dengan institusi medis terdaftar di bawah Undang-Undang Ekspansi Medis Luar Negeri Korea, memastikan layanan yang aman dan terpercaya.",
                service_2_title: "Dukungan Total di Luar Perawatan",
                service_2_desc: "Kami membantu seluruh proses: transportasi, akomodasi, interpretasi profesional, dan produk pasca-perawatan K-Beauty.",
                service_3_title: "Paket Medis yang Disesuaikan",
                service_3_desc: "Kami menawarkan paket khusus yang dioptimalkan sesuai kebutuhan pelanggan, termasuk bedah plastik, dermatologi, dan pemeriksaan kesehatan komprehensif.",
                consult_title: "Konsultasi Profesional Real-Time 1:1 (Obrolan Aman)",
                consult_desc: "Ajukan pertanyaan apa pun yang Anda miliki. Koordinator ahli kami akan memberikan jawaban yang cepat dan akurat.",
                send_button: "Kirim",
                placeholder_input: "Ketik pesan Anda...",
                user_id_label: "ID Pengguna Saat Ini:",
                language_switcher: "Pilih Bahasa",
                cta_consult_button: "Ajukan Konsultasi 1:1",
                chat_empty_message: "Ketik pesan di bawah untuk memulai konsultasi.",
                loading_auth_message: "Sedang mengautentikasi. Mohon tunggu sebentar.",
                footer_copyright: "&copy; 2024 Platform K-TOTAL CARE. Semua hak dilindungi. | Didukung oleh TAELYN MED, LCS Partners, UNICORE Indonesia.",
                footer_address_prefix: "Alamat Platform:",
                // 태린메드 사업 개요 (NEW)
                biz_overview_title: "Tinjauan Bisnis & Visi TAELYN MED",
                biz_overview_p1: "PT TAELYN MED adalah perusahaan profesional yang didirikan untuk menarik dan mengelola pasien asing. Melalui usaha patungan dengan LCS Partners dan UNICORE Indonesia, kami bertindak sebagai jembatan yang menghubungkan pasien di seluruh Asia dengan layanan medis unggul di Korea. Visi kami adalah memimpin pasar pariwisata medis melalui sistem perawatan total yang sistematis berbasis platform IT.",
                market_title: "Target Pasar Utama",
                market_1: "Indonesia: Fokus pasar dan pembentukan pijakan melalui anak perusahaan lokal (PT UNICORE BIZPAY INDONESIA)",
                market_2: "Vietnam/Thailand: Ekspansi bisnis melalui kemitraan dengan agen dan perusahaan perjalanan lokal",
                market_3: "Tiongkok: Masuk dan stabilisasi pasar melalui aliansi strategis dengan agen perjalanan besar",
                market_4: "Amerika Serikat: Ekspansi bisnis jangka panjang ke pasar global",
                service_main_title: "Layanan Utama",
                service_main: "Kami menyediakan 'Layanan Perawatan Total' yang mencakup seluruh perjalanan pasien, termasuk konsultasi waktu nyata berbasis platform IT, pemesanan rumah sakit dan manajemen jadwal, serta layanan penghubung untuk interpretasi, akomodasi, dan transportasi.",
                // 태린메드 고객 신뢰 섹션 (기존)
                trust_title: "Panduan Rincian Kepercayaan Pelanggan TAELYN MED",
                trust_desc: "PT TAELYN MED, operator platform K-TOTAL CARE, memprioritaskan keselamatan dan kepercayaan pelanggan melalui kemitraan resmi dengan institusi medis dan agensi khusus yang terdaftar untuk menarik pasien asing. Anda dapat memeriksa status kemitraan di bawah.",
                trust_hospital_title: "Rumah Sakit Mitra Resmi dan Institusi Khusus",
                trust_hospital_list: [
                    "Klinik Dojaki (Dermatologi, Bedah Plastik)",
                    "Klinik Gigi One Day (Perawatan Gigi)",
                    "Klinik Mata Haneul (Oftalmologi)",
                    "Rumah Sakit A (Pemeriksaan Umum)",
                    "Bedah Plastik B (Bedah Plastik)",
                    "Dermatologi C (Kulit/Estetika)",
                    "Klinik Gigi D (Ortodontik)",
                    "Klinik Obat Tradisional E (Pengobatan Tradisional Korea)",
                    "Pusat Pemeriksaan Kesehatan F (Pemeriksaan Kesehatan Komprehensif)"
                ],
                trust_license_title: "Registrasi dan Sertifikasi Utama",
                trust_license_1: "Registrasi Resmi sebagai Bisnis Penarik Pasien Asing (Kementerian Kesehatan dan Kesejahteraan)",
                trust_license_2: "Penyediaan Layanan Perawatan Total Pariwisata Medis berbasis Platform IT",
                trust_license_3: "Kemitraan Resmi dengan Anak Perusahaan Lokal Luar Negeri (PT UNICORE BIZPAY INDONESIA)",
            },
            'th': {
                platform_name: "แพลตฟอร์ม K-TOTAL CARE",
                subtitle: "บริการ K-Medical และการดูแลครบวงจรในที่เดียว!",
                menu_home: "หน้าแรก",
                menu_about: "ข้อมูลแพลตฟอร์ม",
                menu_services: "บริการหลัก",
                menu_consultation: "ปรึกษา 1:1",
                main_title: "การเริ่มต้นใหม่ของการท่องเที่ยวเชิงการแพทย์ในเกาหลี",
                main_description: "ktotalcare.com คือแพลตฟอร์มการดูแลครบวงจรแบบ All-in-One ที่เชื่อมโยงผู้ป่วยจากเอเชีย (อินโดนีเซีย, เวียดนาม ฯลฯ) เข้ากับสถาบันทางการแพทย์ชั้นนำของเกาหลี",
                section_title_1: "ทำไมต้อง K-TOTAL CARE?",
                service_1_title: "เครือข่ายโรงพยาบาลที่ผ่านการตรวจสอบ",
                service_1_desc: "เราร่วมมือเฉพาะกับสถาบันทางการแพทย์ที่เชี่ยวชาญซึ่งจดทะเบียนภายใต้กฎหมายการขยายการแพทย์ในต่างประเทศของเกาหลี ทำให้มั่นใจได้ว่าบริการมีความปลอดภัยและน่าเชื่อถือ",
                service_2_title: "การสนับสนุนครบวงจรนอกเหนือจากการรักษา",
                service_2_desc: "เราให้ความช่วยเหลือตลอดกระบวนการ: การเดินทาง, ที่พัก, ล่ามมืออาชีพ, และการเชื่อมโยงกับผลิตภัณฑ์ดูแลหลังการรักษา K-Beauty",
                service_3_title: "แพ็คเกจทางการแพทย์ที่ปรับแต่งได้",
                service_3_desc: "เราเสนอแพ็คเกจเฉพาะทางที่เหมาะสมที่สุดที่ปรับให้เข้ากับความต้องการของลูกค้า รวมถึงศัลยกรรมพลาสติก, ผิวหนัง, และการตรวจสุขภาพที่ครอบคลุม",
                consult_title: "การปรึกษาหารือจากผู้เชี่ยวชาญแบบเรียลไทม์ 1:1 (แชทปลอดภัย)",
                consult_desc: "สอบถามคำถามที่คุณมี ผู้ประสานงานผู้เชี่ยวชาญของเราจะให้คำตอบที่รวดเร็วและแม่นยำ",
                send_button: "ส่ง",
                placeholder_input: "พิมพ์ข้อความของคุณ...",
                user_id_label: "ID ผู้ใช้ปัจจุบัน:",
                language_switcher: "เลือกภาษา",
                cta_consult_button: "สมัครเข้ารับการปรึกษา 1:1",
                chat_empty_message: "พิมพ์ข้อความด้านล่างเพื่อเริ่มการปรึกษา.",
                loading_auth_message: "กำลังตรวจสอบสิทธิ์ กรุณารอสักครู่.",
                footer_copyright: "&copy; 2024 แพลตฟอร์ม K-TOTAL CARE. สงวนลิขสิทธิ์ | สนับสนุนโดย TAELYN MED, LCS Partners, UNICORE Indonesia.",
                footer_address_prefix: "ที่อยู่แพลตฟอร์ม:",
                // 태린메드 사업 개요 (NEW)
                biz_overview_title: "ภาพรวมธุรกิจและวิสัยทัศน์ TAELYN MED",
                biz_overview_p1: "บริษัท TAELYN MED จำกัด เป็นบริษัทมืออาชีพที่จัดตั้งขึ้นเพื่อดึงดูดและบริหารจัดการผู้ป่วยชาวต่างชาติ เราทำหน้าที่เป็นสะพานเชื่อมต่อผู้ป่วยทั่วเอเชียเข้ากับบริการทางการแพทย์ที่ยอดเยี่ยมของเกาหลี ผ่านการร่วมทุนกับ LCS Partners และบริษัท UNICORE Indonesia วิสัยทัศน์ของเราคือการเป็นผู้นำตลาดการท่องเที่ยวเชิงการแพทย์ผ่านระบบการดูแลครบวงจรที่เป็นระบบและมีพื้นฐานจากแพลตฟอร์ม IT",
                market_title: "ตลาดเป้าหมายหลัก",
                market_1: "อินโดนีเซีย: เน้นตลาดและการสร้างฐานที่มั่นผ่านบริษัทในเครือท้องถิ่น (PT UNICORE BIZPAY INDONESIA)",
                market_2: "เวียดนาม/ไทย: ขยายธุรกิจผ่านความร่วมมือกับตัวแทนและบริษัทท่องเที่ยวในท้องถิ่น",
                market_3: "จีน: การเข้าสู่ตลาดและสร้างเสถียรภาพผ่านพันธมิตรเชิงกลยุทธ์กับบริษัทท่องเที่ยวขนาดใหญ่",
                market_4: "สหรัฐอเมริกา: การขยายธุรกิจระยะยาวเข้าสู่ตลาดโลก",
                service_main_title: "บริการหลัก",
                service_main: "เราให้บริการ 'Total Care Service' ที่ครอบคลุมทุกขั้นตอนการเดินทางของผู้ป่วย รวมถึงการให้คำปรึกษาแบบเรียลไทม์ผ่านแพลตฟอร์ม IT, การจองโรงพยาบาลและการจัดการตารางเวลา, และบริการเชื่อมโยงล่าม/ที่พัก/การเดินทาง",
                // 태린메드 고객 신뢰 섹션 (기존)
                trust_title: "คู่มือรายละเอียดความไว้วางใจลูกค้า TAELYN MED",
                trust_desc: "TAELYN MED Co., Ltd., ผู้ดำเนินการแพลตฟอร์ม K-TOTAL CARE, ให้ความสำคัญกับความปลอดภัยและความไว้วางใจของลูกค้าเป็นอันดับแรกผ่านความร่วมมืออย่างเป็นทางการกับสถาบันทางการแพทย์และหน่วยงานพิเศษที่ลงทะเบียนสำหรับการดึงดูดผู้ป่วยชาวต่างชาติ คุณสามารถตรวจสอบสถานะความร่วมมือได้ด้านล่าง",
                trust_hospital_title: "โรงพยาบาลพันธมิตรอย่างเป็นทางการและสถาบันผู้เชี่ยวชาญ",
                trust_hospital_list: [
                    "คลินิก Dojaki (ผิวหนัง, ศัลยกรรมพลาสติก)",
                    "คลินิกทันตกรรม One Day (การรักษาทางทันตกรรม)",
                    "คลินิกตา Haneul (จักษุวิทยา)",
                    "โรงพยาบาล A (การตรวจสุขภาพทั่วไป)",
                    "ศัลยกรรมพลาสติก B (ศัลยกรรมพลาสติก)",
                    "ผิวหนัง C (ผิวหนัง/ความงาม)",
                    "ทันตกรรม D (ทันตกรรมจัดฟัน)",
                    "คลินิกแพทย์แผนเกาหลี E (การแพทย์แผนเกาหลี)",
                    "ศูนย์ตรวจสุขภาพ F (การตรวจสุขภาพอย่างครอบคลุม)"
                ],
                trust_license_title: "การลงทะเบียนและการรับรองที่สำคัญ",
                trust_license_1: "การลงทะเบียนอย่างเป็นทางการเป็นธุรกิจดึงดูดผู้ป่วยชาวต่างชาติ (กระทรวงสาธารณสุขและสวัสดิการ)",
                trust_license_2: "การให้บริการดูแลครบวงจรด้านการท่องเที่ยวเชิงการแพทย์ด้วยแพลตฟอร์ม IT",
                trust_license_3: "การเป็นพันธมิตรอย่างเป็นทางการกับบริษัทในเครือท้องถิ่นในต่างประเทศ (PT UNICORE BIZPAY INDONESIA)",
            },
            'vi': {
                platform_name: "Nền tảng K-TOTAL CARE",
                subtitle: "Dịch vụ Y tế K-Medical và Chăm sóc Toàn diện tại một nơi!",
                menu_home: "Trang chủ",
                menu_about: "Giới thiệu Nền tảng",
                menu_services: "Dịch vụ Chính",
                menu_consultation: "Tư vấn 1:1",
                main_title: "Khởi đầu mới của Du lịch Y tế Hàn Quốc",
                main_description: "ktotalcare.com là nền tảng Chăm sóc Toàn diện All-in-One, kết nối bệnh nhân từ Châu Á (Indonesia, Việt Nam, v.v.) với các cơ sở y tế xuất sắc của Hàn Quốc.",
                section_title_1: "Tại sao chọn K-TOTAL CARE?",
                service_1_title: "Mạng lưới bệnh viện đã được xác minh",
                service_1_desc: "Chúng tôi chỉ hợp tác với các cơ sở y tế chuyên biệt đã đăng ký theo Luật Mở rộng Y tế ra nước ngoài của Hàn Quốc, đảm bảo dịch vụ an toàn và đáng tin cậy.",
                service_2_title: "Hỗ trợ Toàn diện ngoài điều trị",
                service_2_desc: "Chúng tôi hỗ trợ toàn bộ quá trình: giao thông, chỗ ở, phiên dịch chuyên nghiệp và liên kết với các sản phẩm chăm sóc hậu phẫu K-Beauty.",
                service_3_title: "Gói Dịch vụ Y tế Tùy chỉnh",
                service_3_desc: "Chúng tôi cung cấp các gói dịch vụ chuyên biệt tối ưu, phù hợp với nhu cầu của khách hàng, bao gồm phẫu thuật thẩm mỹ, da liễu và kiểm tra sức khỏe toàn diện.",
                consult_title: "Tư vấn Chuyên nghiệp Thời gian Thực 1:1 (Trò chuyện An toàn)",
                consult_desc: "Hãy hỏi bất kỳ câu hỏi nào bạn có. Điều phối viên chuyên gia của chúng tôi sẽ cung cấp câu trả lời nhanh chóng và chính xác.",
                send_button: "Gửi",
                placeholder_input: "Nhập tin nhắn của bạn...",
                user_id_label: "ID Người dùng Hiện tại:",
                language_switcher: "Chọn Ngôn ngữ",
                cta_consult_button: "Đăng ký Tư vấn 1:1",
                chat_empty_message: "Nhập tin nhắn dưới đây để bắt đầu tư vấn.",
                loading_auth_message: "Đang xác thực. Vui lòng đợi trong giây lát.",
                footer_copyright: "&copy; 2024 Nền tảng K-TOTAL CARE. Bảo lưu mọi quyền. | Được cung cấp bởi TAELYN MED, LCS Partners, UNICORE Indonesia.",
                footer_address_prefix: "Địa chỉ Nền tảng:",
                // 태린메드 사업 개요 (NEW)
                biz_overview_title: "Tổng quan & Tầm nhìn Kinh doanh TAELYN MED",
                biz_overview_p1: "TAELYN MED là pháp nhân chuyên nghiệp được thành lập để thu hút và quản lý bệnh nhân nước ngoài. Thông qua liên doanh với LCS Partners và UNICORE Indonesia, chúng tôi đóng vai trò cầu nối, kết nối bệnh nhân trên khắp Châu Á với các dịch vụ y tế vượt trội của Hàn Quốc. Tầm nhìn của chúng tôi là dẫn đầu thị trường du lịch y tế thông qua hệ thống chăm sóc toàn diện, có hệ thống dựa trên nền tảng CNTT.",
                market_title: "Thị trường Mục tiêu Chính",
                market_1: "Indonesia: Tập trung thị trường và thiết lập chỗ đứng thông qua công ty con tại địa phương (PT UNICORE BIZPAY INDONESIA)",
                market_2: "Việt Nam/Thái Lan: Mở rộng kinh doanh thông qua quan hệ đối tác với các đại lý và công ty du lịch địa phương",
                market_3: "Trung Quốc: Gia nhập và ổn định thị trường thông qua liên minh chiến lược với các công ty du lịch lớn",
                market_4: "Hoa Kỳ: Mở rộng kinh doanh dài hạn vào thị trường toàn cầu",
                service_main_title: "Dịch vụ Chính",
                service_main: "Chúng tôi cung cấp 'Dịch vụ Chăm sóc Toàn diện' chịu trách nhiệm cho toàn bộ hành trình của bệnh nhân, bao gồm tư vấn trực tuyến trên nền tảng CNTT, đặt lịch bệnh viện và quản lý lịch trình, cùng với các dịch vụ liên kết về phiên dịch, chỗ ở và giao thông.",
                // 태린메드 고객 신뢰 섹션 (기존)
                trust_title: "Hướng dẫn Chi tiết Tin cậy Khách hàng TAELYN MED",
                trust_desc: "TAELYN MED Co., Ltd., đơn vị vận hành nền tảng K-TOTAL CARE, ưu tiên sự an toàn và tin cậy của khách hàng thông qua quan hệ đối tác chính thức với các cơ sở y tế và cơ quan chuyên môn đã đăng ký để thu hút bệnh nhân nước ngoài. Bạn có thể kiểm tra trạng thái đối tác dưới đây.",
                trust_hospital_title: "Bệnh viện Đối tác Chính thức và Cơ sở Chuyên môn",
                trust_hospital_list: [
                    "Phòng khám Dojaki (Da liễu, Phẫu thuật thẩm mỹ)",
                    "Nha khoa One Day (Điều trị nha khoa)",
                    "Phòng khám Mắt Haneul (Nhãn khoa)",
                    "Bệnh viện A (Kiểm tra sức khỏe tổng quát)",
                    "Phẫu thuật Thẩm mỹ B (Phẫu thuật thẩm mỹ)",
                    "Da liễu C (Da/Thẩm mỹ)",
                    "Nha khoa D (Chỉnh nha)",
                    "Phòng khám Đông y E (Đông y Hàn Quốc)",
                    "Trung tâm Kiểm tra Sức khỏe F (Kiểm tra sức khỏe toàn diện)"
                ],
                trust_license_title: "Đăng ký và Chứng nhận Chính",
                trust_license_1: "Đăng ký Chính thức là Doanh nghiệp Thu hút Bệnh nhân Nước ngoài (Bộ Y tế và Phúc lợi)",
                trust_license_2: "Cung cấp Dịch vụ Chăm sóc Toàn diện Du lịch Y tế dựa trên Nền tảng CNTT",
                trust_license_3: "Quan hệ Đối tác Chính thức với Công ty Con tại Nước ngoài (PT UNICORE BIZPAY INDONESIA)",
            },
            'ja': {
                platform_name: "K-TOTAL CARE プラットフォーム",
                subtitle: "K-医療とトータルケアサービスを一つに！",
                menu_home: "ホーム",
                menu_about: "プラットフォーム紹介",
                menu_services: "主要サービス",
                menu_consultation: "1対1相談",
                main_title: "韓国医療観光の新たなスタート",
                main_description: "ktotalcare.com は、インドネシア、ベトナムなどのアジアの患者と韓国の優れた医療機関を結びつけるオールインワントータルケアプラットフォームです。",
                section_title_1: "なぜ K-TOTAL CARE なのか？",
                service_1_title: "検証された病院ネットワーク",
                service_1_desc: "医療海外進出法に登録された専門医療機関のみと提携し、安全で信頼できるサービスを提供します。",
                service_2_title: "診療以外のトータルサポート",
                service_2_desc: "交通、宿泊、専門通訳、K-ビューティーのアフターケア製品連携まで、全ての過程をサポートします。",
                service_3_title: "オーダーメイド医療パッケージ",
                service_3_desc: "整形、皮膚、健康診断など、お客様のニーズに合わせた最適な特化商品パッケージを提供します。",
                consult_title: "1対1 リアルタイム専門相談 (安全チャット)",
                consult_desc: "ご不明な点はお問い合わせください。専門のコーディネーターが迅速かつ正確にお答えします。",
                send_button: "送信",
                placeholder_input: "メッセージを入力してください...",
                user_id_label: "現在のユーザーID:",
                language_switcher: "言語選択",
                cta_consult_button: "1対1相談を申し込む",
                chat_empty_message: "下にメッセージを入力して相談を開始してください。",
                loading_auth_message: "認証中です。しばらくお待ちください。",
                footer_copyright: "&copy; 2024 K-TOTAL CARE Platform. All rights reserved. | Powered by TAELYN MED, LCS Partners, UNICORE Indonesia.",
                footer_address_prefix: "プラットフォームアドレス:",
                // 태린메드 사업 개요 (NEW)
                biz_overview_title: "㈱TAELYN MED 事業概要およびビジョン",
                biz_overview_p1: "㈱TAELYN MEDは、外国人患者の誘致および管理のために設立された専門法人です。LCS Partners、UNICOREインドネシア法人との共同事業を通じて、アジア全域の患者を国内の優れた医療サービスに繋ぐ架け橋の役割を果たしています。ITプラットフォームを基盤とした体系的なトータルケアシステムを通じて、医療観光市場をリードすることが私たちのビジョンです。",
                market_title: "主要ターゲット市場",
                market_1: "インドネシア: 現地法人(PT UNICORE BIZPAY INDONESIA)を通じた市場集中および拠点確保",
                market_2: "ベトナム/タイ: 現地エージェンシーおよび旅行会社との提携による事業拡大",
                market_3: "中国: 大手旅行会社との戦略的提携による市場進出および安定化",
                market_4: "米国: 長期的に事業領域を拡大し、グローバル市場へ進出",
                service_main_title: "主要サービス",
                service_main: "ITプラットフォームを基盤としたリアルタイム相談、病院予約およびスケジュール管理、通訳・宿泊・交通連携サービスなど、患者の全行程を担う「トータルケアサービス」を提供します。",
                // 태린메드 고객 신뢰 섹션 (기존)
                trust_title: "㈱TAELYN MED 顧客信頼詳細案内",
                trust_desc: "K-TOTAL CARE プラットフォーム運営会社である㈱TAELYN MED は、外国人患者誘致登録医療機関および専門機関との公式提携を通じて、お客様の安全と信頼を最優先にしています。以下の提携状況をご確認いただけます。",
                trust_hospital_title: "公式提携病院および専門機関",
                trust_hospital_list: [
                    "Dojaki クリニック (皮膚、整形)",
                    "One Day 歯科 (歯科診療)",
                    "Haneul 眼科 (眼科診療)",
                    "A 病院 (総合健診)",
                    "B 整形外科 (整形手術)",
                    "C 皮膚科 (皮膚/美容)",
                    "D 歯科 (歯科矯正)",
                    "E 漢方医院 (漢方診療)",
                    "F 健康診断センター (総合健康診断)"
                ],
                trust_license_title: "主要登録および認証状況",
                trust_license_1: "外国人患者誘致業 正式登録 (保健福祉部)",
                trust_license_2: "ITプラットフォーム基盤の医療観光トータルケアサービス提供",
                trust_license_3: "海外現地法人 (PT UNICORE BIZPAY INDONESIA)との公式パートナーシップ",
            }
        };

        let currentLang = 'ko';

        // 새로운 헬퍼 함수: 다른 스크립트 블록(module)에서 번역 텍스트를 가져오기 위해 전역으로 노출합니다.
        window.getL10nText = function(key) {
            return translations[currentLang]?.[key] || `[Translation Error: ${key}]`;
        }

        // 제휴 병원 리스트를 동적으로 렌더링하는 함수
        function renderPartnerList() {
            const listElement = document.getElementById('partner-hospital-list');
            if (!listElement) return;

            const listData = translations[currentLang]?.trust_hospital_list || [];
            listElement.innerHTML = ''; // Clear existing list

            listData.forEach(item => {
                const li = document.createElement('li');
                li.className = 'flex items-center space-x-2 text-gray-700 hover:text-indigo-600 transition duration-150';
                li.innerHTML = `<i class="fas fa-check-circle text-indigo-500 text-sm"></i> <span>${item}</span>`;
                listElement.appendChild(li);
            });
        }


        function updateTexts() {
            const t = translations[currentLang];
            document.querySelectorAll('[data-l10n-key]').forEach(el => {
                const key = el.getAttribute('data-l10n-key');
                if (t[key]) {
                    // Check if the element is an input/textarea (like chat-input)
                    if (el.tagName === 'INPUT' || el.tagName === 'TEXTAREA') {
                        el.placeholder = t[key];
                    } else if (key === 'footer_copyright') {
                        // HTML content for copyright
                        el.innerHTML = t[key];
                    } else if (key === 'footer_address_prefix') {
                        // Address prefix
                        el.textContent = t[key];
                    }
                     else {
                        // For regular elements (div, span, button, a)
                        el.textContent = t[key];
                    }
                }
            });
            
            // 언어 전환 시 빈 채팅창 메시지 및 파트너 리스트도 업데이트
            const chatMessagesDiv = document.getElementById('chat-messages');
            if (chatMessagesDiv && chatMessagesDiv.children.length === 1 && 
                chatMessagesDiv.children[0].textContent.includes(translations['ko']['chat_empty_message'].substring(0, 5))) {
                
                const emptyMessageText = window.getL10nText('chat_empty_message');
                chatMessagesDiv.innerHTML = `<div class="p-4 text-center text-gray-400">${emptyMessageText}</div>`;
            }
            
            // 파트너 리스트 업데이트 호출
            renderPartnerList(); 


            // Update language switcher appearance for all buttons
            SUPPORTED_LANGS.forEach(lang => {
                const button = document.getElementById(`lang-${lang}`);
                if (button) {
                    button.classList.toggle('bg-gray-700', currentLang === lang);
                    button.classList.toggle('text-white', currentLang === lang);
                    button.classList.toggle('bg-gray-200', currentLang !== lang);
                }
            });
        }

        function switchLanguage(lang) {
            currentLang = lang;
            updateTexts();
        }

        document.addEventListener('DOMContentLoaded', () => {
            // Set initial language to Korean
            switchLanguage('ko');

            // Attach event listeners to all language buttons
            SUPPORTED_LANGS.forEach(lang => {
                const button = document.getElementById(`lang-${lang}`);
                if (button) {
                    button.addEventListener('click', () => switchLanguage(lang));
                }
            });

            // Attach event listener for chat input (Enter key)
            const chatInput = document.getElementById('chat-input');
            if (chatInput) {
                chatInput.addEventListener('keypress', function(e) {
                    if (e.key === 'Enter' && !e.shiftKey) {
                        e.preventDefault();
                        window.sendMessage();
                    }
                });
            }
        });
    </script>


    <!-- Header Section -->
    <header class="header-bg shadow-lg">
        <div class="container-fluid flex justify-between items-center py-4">
            <!-- Logo/Platform Name -->
            <h1 class="text-3xl font-extrabold text-white">
                <span data-l10n-key="platform_name">K-TOTAL CARE 플랫폼</span>
            </h1>

            <!-- Language Switcher & Google Translate (Updated with 7 languages) -->
            <div class="flex items-center space-x-4">
                <div class="flex space-x-1 p-1 bg-gray-500 rounded-full text-xs md:text-sm">
                    <button id="lang-ko" class="px-2 py-1 font-semibold rounded-full hover:bg-gray-600 transition duration-150 text-white">KO</button>
                    <button id="lang-en" class="px-2 py-1 font-semibold rounded-full hover:bg-gray-600 transition duration-150 text-white">EN</button>
                    <button id="lang-zh" class="px-2 py-1 font-semibold rounded-full hover:bg-gray-600 transition duration-150 text-white">ZH</button>
                    <button id="lang-id" class="px-2 py-1 font-semibold rounded-full hover:bg-gray-600 transition duration-150 text-white">ID</button>
                    <button id="lang-th" class="px-2 py-1 font-semibold rounded-full hover:bg-gray-600 transition duration-150 text-white">TH</button>
                    <button id="lang-vi" class="px-2 py-1 font-semibold rounded-full hover:bg-gray-600 transition duration-150 text-white">VI</button>
                    <button id="lang-ja" class="px-2 py-1 font-semibold rounded-full hover:bg-gray-600 transition duration-150 text-white">JA</button>
                </div>
                <!-- Google Translate Dropdown -->
                <div id="google_translate_element" class="text-white text-sm"></div>
            </div>
        </div>
        <!-- Navigation -->
        <nav class="container-fluid py-3">
            <ul class="flex space-x-6 text-white font-medium">
                <li><a href="#home" class="hover:text-red-300 transition duration-150" data-l10n-key="menu_home">홈</a></li>
                <li><a href="#about" class="hover:text-red-300 transition duration-150" data-l10n-key="menu_about">플랫폼 소개</a></li>
                <li><a href="#services" class="hover:text-red-300 transition duration-150" data-l10n-key="menu_services">주요 서비스</a></li>
                <li><a href="#consultation" class="hover:text-red-300 transition duration-150" data-l10n-key="menu_consultation">1:1 상담</a></li>
            </ul>
        </nav>
    </header>

    <main class="py-10">

        <!-- Main Hero Section -->
        <section id="home" class="container-fluid text-center mb-20">
            <div class="bg-white p-12 rounded-2xl card">
                <h2 class="text-5xl font-extrabold text-gray-800 mb-4" data-l10n-key="main_title">대한민국 의료 관광의 새로운 시작</h2>
                <p class="text-xl text-gray-600 mb-8" data-l10n-key="main_description">ktotalcare.com은 인도네시아, 베트남 등 아시아 환자들을 한국의 우수한 의료기관과 연결하는 올인원(All-in-One) 토탈 케어 플랫폼입니다.</p>
                <!-- CTA 버튼에 data-l10n-key 적용 -->
                <a href="#consultation" class="cta-button inline-block px-8 py-3 text-lg font-bold text-white rounded-full shadow-lg">
                    <i class="fas fa-comments mr-2"></i> <span data-l10n-key="cta_consult_button">1:1 상담 신청하기</span>
                </a>
            </div>
        </section>

        <!-- Services Section -->
        <section id="services" class="container-fluid mb-20">
            <h3 class="text-4xl font-bold text-center text-gray-800 mb-12" data-l10n-key="section_title_1">왜 K-TOTAL CARE인가?</h3>
            <div class="grid grid-cols-1 md:grid-cols-3 gap-8">
                <!-- Service Card 1 -->
                <div class="card bg-white p-6 rounded-xl text-center">
                    <div class="text-4xl text-indigo-600 mb-4"><i class="fas fa-shield-alt"></i></div>
                    <h4 class="text-xl font-semibold mb-3" data-l10n-key="service_1_title">검증된 병원 네트워크</h4>
                    <p class="text-gray-600" data-l10n-key="service_1_desc">의료해외진출법에 등록된 전문 의료기관만 제휴하여 안전하고 신뢰할 수 있는 서비스를 제공합니다.</p>
                </div>
                <!-- Service Card 2 -->
                <div class="card bg-white p-6 rounded-xl text-center">
                    <div class="text-4xl text-indigo-600 mb-4"><i class="fas fa-globe-americas"></i></div>
                    <h4 class="text-xl font-semibold mb-3" data-l10n-key="service_2_title">진료 외 토탈 지원</h4>
                    <p class="text-gray-600" data-l10n-key="service_2_desc">교통, 숙박, 전문 통역, K-뷰티 사후 관리 제품 연계까지 모든 과정을 지원합니다.</p>
                </div>
                <!-- Service Card 3 -->
                <div class="card bg-white p-6 rounded-xl text-center">
                    <div class="text-4xl text-indigo-600 mb-4"><i class="fas fa-heartbeat"></i></div>
                    <h4 class="text-xl font-semibold mb-3" data-l10n-key="service_3_title">맞춤형 의료 패키지</h4>
                    <p class="text-gray-600" data-l10n-key="service_3_desc">성형, 피부, 건강검진 등 고객의 니즈에 맞춘 최적의 특화 상품 패키지를 제공합니다.</p>
                </div>
            </div>
        </section>

        <!-- About Us Section: TAELYN MED Business Overview and Trust Details -->
        <section id="about" class="container-fluid mb-20">
            
            <!-- TAELYN MED Business Overview Section (NEW) -->
            <div class="bg-white p-6 md:p-10 rounded-2xl card mb-10 border-t-4 border-indigo-500">
                <h3 class="text-3xl font-bold text-gray-800 mb-6" data-l10n-key="biz_overview_title">㈜태린메드 사업 개요 및 비전</h3>
                <p class="text-lg text-gray-600 mb-8 leading-relaxed" data-l10n-key="biz_overview_p1">㈜태린메드는 외국인 환자 유치 및 관리를 위해 설립된 전문 법인입니다. 우리는 엘씨에스파트너스, 유니코아 인도네시아 법인과의 공동 사업을 통해 아시아 전역의 환자들에게 국내의 우수한 의료 서비스를 연결하는 교두보 역할을 수행하고 있습니다. IT 플랫폼 기반의 체계적인 토탈 케어 시스템을 통해 의료 관광 시장을 선도하는 것이 우리의 비전입니다.</p>

                <!-- Key Target Markets -->
                <div class="mb-8">
                    <h4 class="text-2xl font-semibold text-gray-700 mb-4 border-b pb-2" data-l10n-key="market_title">주요 타겟 시장</h4>
                    <ul class="space-y-3 text-gray-700 text-base">
                        <li class="flex items-start space-x-3"><i class="fas fa-map-marker-alt text-red-500 mt-1 flex-shrink-0"></i> <span data-l10n-key="market_1">인도네시아: 현지 법인(PT UNICORE BIZPAY INDONESIA)을 통한 시장 집중 및 거점 확보</span></li>
                        <li class="flex items-start space-x-3"><i class="fas fa-map-marker-alt text-red-500 mt-1 flex-shrink-0"></i> <span data-l10n-key="market_2">베트남/태국: 현지 에이전시 및 여행사와의 제휴를 통한 사업 확장</span></li>
                        <li class="flex items-start space-x-3"><i class="fas fa-map-marker-alt text-red-500 mt-1 flex-shrink-0"></i> <span data-l10n-key="market_3">중국: 대형 여행사와의 전략적 제휴를 통한 시장 진출 및 안정화</span></li>
                        <li class="flex items-start space-x-3"><i class="fas fa-map-marker-alt text-red-500 mt-1 flex-shrink-0"></i> <span data-l10n-key="market_4">미국: 장기적으로 사업 영역을 확장하여 글로벌 시장 진출</span></li>
                    </ul>
                </div>

                <!-- Main Services Offered -->
                <div>
                    <h4 class="text-2xl font-semibold text-gray-700 mb-4 border-b pb-2" data-l10n-key="service_main_title">주요 서비스</h4>
                    <p class="text-gray-700 leading-relaxed text-base" data-l10n-key="service_main">IT 플랫폼 기반의 실시간 상담, 병원 예약 및 스케줄 관리, 통역/숙소/교통 연계 서비스 등 환자의 전 여정을 책임지는 '토탈 케어 서비스'를 제공합니다.</p>
                </div>
            </div>

            <!-- TAELYN MED Trust Section (Existing Section - Modified styling for distinction) -->
            <div class="bg-white p-6 md:p-10 rounded-2xl card border-t-4 border-green-500">
                <h3 class="text-3xl font-bold text-gray-800 mb-4" data-l10n-key="trust_title">주식회사 태린메드 (TAELYN MED) 고객 신뢰 상세 안내</h3>
                <p class="text-gray-600 mb-8 leading-relaxed" data-l10n-key="trust_desc">K-TOTAL CARE 플랫폼 운영사인 (주)태린메드는 외국인환자유치 등록 의료기관 및 전문 기관과의 공식 제휴를 통해 고객의 안전과 신뢰를 최우선으로 합니다. 아래 제휴 현황을 확인하실 수 있습니다.</p>

                <div class="grid grid-cols-1 lg:grid-cols-2 gap-10">
                    <!-- 제휴 병원 리스트 -->
                    <div>
                        <h4 class="text-xl font-semibold text-gray-700 mb-4 border-b pb-2" data-l10n-key="trust_hospital_title">공식 제휴 병원 및 전문기관</h4>
                        <ul id="partner-hospital-list" class="partner-list space-y-2">
                            <!-- JS에 의해 동적으로 채워집니다 -->
                        </ul>
                    </div>

                    <!-- 등록 및 인증 현황 -->
                    <div>
                        <h4 class="text-xl font-semibold text-gray-700 mb-4 border-b pb-2" data-l10n-key="trust_license_title">주요 등록 및 인증 현황</h4>
                        <ul class="space-y-3">
                            <li class="flex items-start space-x-3 text-gray-700">
                                <i class="fas fa-certificate text-green-500 mt-1 flex-shrink-0"></i>
                                <span data-l10n-key="trust_license_1">외국인환자유치업 정식 등록 (보건복지부)</span>
                            </li>
                            <li class="flex items-start space-x-3 text-gray-700">
                                <i class="fas fa-laptop-medical text-green-500 mt-1 flex-shrink-0"></i>
                                <span data-l10n-key="trust_license_2">IT 플랫폼 기반 의료 관광 토탈 케어 서비스 제공</span>
                            </li>
                            <li class="flex items-start space-x-3 text-gray-700">
                                <i class="fas fa-hands-helping text-green-500 mt-1 flex-shrink-0"></i>
                                <span data-l10n-key="trust_license_3">해외 현지 법인 (PT UNICORE BIZPAY INDONESIA)과의 공식 파트너십</span>
                            </li>
                        </ul>
                    </div>
                </div>
            </div>
        </section>

        
        <!-- 1:1 Consultation Section (Chat App) -->
        <section id="consultation" class="container-fluid">
            <div class="bg-white p-6 md:p-10 rounded-2xl card">
                <h3 class="text-3xl font-bold text-gray-800 mb-3" data-l10n-key="consult_title">1:1 실시간 전문 상담 (Secure Chat)</h3>
                <p class="text-gray-600 mb-6" data-l10n-key="consult_desc">궁금한 점을 문의하세요. 전문 코디네이터가 신속하고 정확하게 답변해 드립니다.</p>
                
                <div class="flex flex-col h-[500px] bg-gray-50 rounded-lg border border-gray-200">
                    <!-- Chat Messages Area -->
                    <div id="chat-messages" class="flex-grow p-4 overflow-y-auto space-y-4">
                        <!-- 이 부분은 JS에 의해 동적으로 채워집니다. -->
                    </div>
                    
                    <!-- Chat Input Area -->
                    <div class="p-4 border-t border-gray-200 flex items-center bg-white">
                        <!-- data-l10n-key를 사용하여 placeholder 번역 -->
                        <textarea id="chat-input" rows="1" class="flex-grow p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500 resize-none overflow-hidden" data-l10n-key="placeholder_input"></textarea>
                        <button onclick="sendMessage()" class="cta-button ml-3 px-5 py-2 rounded-lg text-white font-semibold">
                            <span data-l10n-key="send_button">전송</span> <i class="fas fa-paper-plane ml-1"></i>
                        </button>
                    </div>
                </div>

                <!-- User ID Display (for debugging and identification) -->
                <div class="text-xs text-gray-500 mt-4">
                    <span data-l10n-key="user_id_label">현재 사용자 ID:</span> <span id="user-id-display" class="font-mono text-gray-700 break-all">ID 로딩 중...</span>
                </div>
            </div>
        </section>

    </main>

    <!-- Footer Section -->
    <footer class="header-bg py-6 mt-10">
        <div class="container-fluid text-center text-gray-400 text-sm">
            <!-- data-l10n-key를 사용하여 저작권 및 주소 접두사 번역 -->
            <p data-l10n-key="footer_copyright" class="mb-1">&copy; 2024 K-TOTAL CARE Platform. All rights reserved. | Powered by TAELYN MED, LCS Partners, UNICORE Indonesia.</p>
            <!-- 주소는 변하지 않으므로 접두사만 번역 처리하고 주소는 고정 -->
            <p class="mt-2">
                <span data-l10n-key="footer_address_prefix">플랫폼 주소:</span>
                <span class="font-mono text-gray-300">https://chespro-director.github.io/ktotalcare/</span>
            </p>
        </div>
    </footer>

</body>
</html>
