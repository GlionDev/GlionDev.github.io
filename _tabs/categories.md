---
layout: categories
icon: fas fa-stream
order: 1
---

<style>
  /* 1. 마우스 커서 제한(🚫) 강제 해제 */
  /* html body를 앞에 붙여서 우선순위를 테마보다 높임 */
  html body .collapse-toggle,
  html body .collapse-toggle.disabled {
    cursor: pointer !important;       /* 무조건 손가락 모양 */
    pointer-events: auto !important;  /* 무조건 클릭 가능 */
    opacity: 1 !important;            /* 흐릿함 제거 */
    color: var(--heading-color) !important; /* 글자색 정상화 */
  }

  /* 2. 카테고리 내용물 강제로 펼치기 */
  html body #category-list .collapse {
    display: block !important;
    height: auto !important;
    visibility: visible !important;
  }

  /* 3. 화살표 아이콘 방향 아래로 고정 */
  html body #category-list .fa-angle-down {
    transform: rotate(0deg) !important;
  }
</style>

<script>
  // 페이지 로드 완료 시 뿐만 아니라, 윈도우 전체 로딩 후에도 실행
  window.addEventListener('load', function() {
    
    function forceUnlock() {
      // 모든 카테고리 토글 버튼 가져오기
      const triggers = document.querySelectorAll('.collapse-toggle');
      
      triggers.forEach(trigger => {
        // 테마가 붙인 'disabled' 클래스를 제거
        trigger.classList.remove('disabled');
        // 스크린 리더를 위해 '펼쳐짐' 상태로 표시
        trigger.setAttribute('aria-expanded', 'true');
      });

      // 내용물(.collapse)에도 'show' 클래스 붙이기 (Bootstrap 호환성)
      const collapses = document.querySelectorAll('.collapse');
      collapses.forEach(collapse => {
        collapse.classList.add('show');
      });
    }

    // 1차 실행
    forceUnlock();
    
    // 테마 JS가 늦게 로드되어 덮어쓰는 것을 방지하기 위해 0.5초 간격으로 재시도
    setTimeout(forceUnlock, 500);
    setTimeout(forceUnlock, 1500);
  });
</script>