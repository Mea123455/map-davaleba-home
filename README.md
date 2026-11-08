<!DOCTYPE html>
<html lang="ka">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>თბილისის ინტერაქტიული რუკა</title>

  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
        integrity="sha256-p4NxAoJBhIIN+hmNHrzRCf9tD/miZyoHS5obTRR9BMY=" crossorigin="">

  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      font-family: "Segoe UI", "Noto Sans Georgian", sans-serif;
      background: #f4f1ea;
      color: #2b2b2b;
    }

    header {
      background: #7a1f2b;
      color: #fff;
      padding: 14px 24px;
    }
    header h1 { font-size: 22px; }
    header p  { font-size: 14px; opacity: .85; margin-top: 4px; }

    .container {
      display: flex;
      height: calc(100vh - 72px);
    }

    #map {
      flex: 1;
      height: 100%;
    }

    #sidebar {
      width: 340px;
      background: #fff;
      border-left: 1px solid #ddd;
      padding: 20px;
      overflow-y: auto;
    }

    #info-title {
      font-size: 20px;
      color: #7a1f2b;
      margin-bottom: 6px;
    }
    #info-category {
      display: inline-block;
      font-size: 12px;
      padding: 3px 10px;
      border-radius: 12px;
      color: #fff;
      margin-bottom: 12px;
    }
    #info-description {
      font-size: 15px;
      line-height: 1.6;
    }
    .placeholder { color: #888; font-style: italic; }

    .legend {
      margin-top: 28px;
      padding-top: 16px;
      border-top: 1px solid #eee;
    }
    .legend h3 { font-size: 15px; margin-bottom: 10px; }
    .legend-item {
      display: flex;
      align-items: center;
      gap: 10px;
      margin-bottom: 8px;
      font-size: 14px;
    }

    .custom-icon {
      width: 34px;
      height: 34px;
      border-radius: 50% 50% 50% 0;
      transform: rotate(-45deg);
      display: flex;
      align-items: center;
      justify-content: center;
      border: 2px solid #fff;
      box-shadow: 0 2px 6px rgba(0,0,0,.4);
    }
    .custom-icon span {
      transform: rotate(45deg);
      font-size: 16px;
    }
    .icon-history { background: #b5462f; }
    .icon-museum  { background: #2f5fb5; }
    .icon-park    { background: #2f8f4e; }

    .active-marker .custom-icon {
      border-color: #ffd54a;
      box-shadow: 0 0 0 4px rgba(255, 213, 74, .6);
    }

    .legend .custom-icon { width: 26px; height: 26px; }
    .legend .custom-icon span { font-size: 12px; }

    @media (max-width: 768px) {
      .container { flex-direction: column; height: auto; }
      #map { height: 60vh; }
      #sidebar { width: 100%; border-left: none; border-top: 1px solid #ddd; }
    }
  </style>
</head>
<body>

<header>
  <h1>თბილისის ღირსშესანიშნაობები</h1>
  <p>დააკლიკეთ მარკერს, რომ ნახოთ ადგილის დასახელება და აღწერა</p>
</header>

<div class="container">
  <div id="map"></div>

  <aside id="sidebar">
    <h2 id="info-title">აირჩიეთ ადგილი</h2>
    <span id="info-category" style="display:none"></span>
    <p id="info-description" class="placeholder">
      რუკაზე რომელიმე მარკერზე დაკლიკების შემდეგ აქ გამოჩნდება მისი დასახელება და მოკლე აღწერა.
    </p>

    <div class="legend">
      <h3>ლეგენდა</h3>
      <div class="legend-item">
        <div class="custom-icon icon-history"><span>🏰</span></div> ისტორიული ძეგლი
      </div>
      <div class="legend-item">
        <div class="custom-icon icon-museum"><span>🏛️</span></div> მუზეუმი
      </div>
      <div class="legend-item">
        <div class="custom-icon icon-park"><span>🌳</span></div> პარკი / ბუნება
      </div>
    </div>
  </aside>
</div>

<!-- Leaflet JS -->
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        integrity="sha256-20nQCchB9co0qIjJZRGuk2/Z9VM+kNiyxNV1lvTlZBo=" crossorigin=""></script>

<script>
  // 1. Создание карты с центром на Тбилиси
  const map = L.map('map').setView([41.6938, 44.8015], 14);

  // 2. Слой тайлов OpenStreetMap
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
  }).addTo(map);

  // 3. Три разные иконки
  function makeIcon(cssClass, emoji) {
    return L.divIcon({
      className: '',
      html: `<div class="custom-icon ${cssClass}"><span>${emoji}</span></div>`,
      iconSize: [34, 34],
      iconAnchor: [17, 34],
      popupAnchor: [0, -34]
    });
  }

  const icons = {
    history: makeIcon('icon-history', '🏰'),
    museum:  makeIcon('icon-museum',  '🏛️'),
    park:    makeIcon('icon-park',    '🌳')
  };

  const categories = {
    history: { label: 'ისტორიული ძეგლი', color: '#b5462f' },
    museum:  { label: 'მუზეუმი',          color: '#2f5fb5' },
    park:    { label: 'პარკი / ბუნება',    color: '#2f8f4e' }
  };

  // 4. Данные о местах (10 маркеров)
  const places = [
    {
      name: 'ნარიყალა',
      type: 'history',
      coords: [41.6878, 44.8086],
      description: 'IV საუკუნით დათარიღებული ციხე-სიმაგრე, რომელიც სოლოლაკის ქედზე დგას და ძველ თბილისს გადაჰყურებს. ციხის ტერიტორიაზე აღდგენილია წმინდა ნიკოლოზის ეკლესია.'
    },
    {
      name: 'წმინდა სამების საკათედრო ტაძარი',
      type: 'history',
      coords: [41.6973, 44.8167],
      description: 'საქართველოს მართლმადიდებელი ეკლესიის მთავარი ტაძარი, აშენდა 1995–2004 წლებში ელიას მთაზე. ერთ-ერთი უდიდესი მართლმადიდებლური ტაძარია მსოფლიოში.'
    },
    {
      name: 'მეტეხის ღვთისმშობლის ტაძარი',
      type: 'history',
      coords: [41.6893, 44.8113],
      description: 'XIII საუკუნის ტაძარი მტკვრის მაღალ კლდოვან ნაპირზე. მის წინ დგას თბილისის დამაარსებლის, ვახტანგ გორგასლის ცხენოსანი ქანდაკება.'
    },
    {
      name: 'სიონის ტაძარი',
      type: 'history',
      coords: [41.6915, 44.8075],
      description: 'ძველ თბილისში მდებარე ისტორიული ტაძარი, რომლის პირველი ნაგებობა VI–VII საუკუნეებით თარიღდება. აქ ინახება წმინდა ნინოს ჯვარი.'
    },
    {
      name: 'საქართველოს ეროვნული მუზეუმი',
      type: 'museum',
      coords: [41.6966, 44.7997],
      description: 'სიმონ ჯანაშიას სახელობის მუზეუმი რუსთაველის გამზირზე. ცნობილია ოქროს საგანძურის გამოფენით და დმანისში აღმოჩენილი უძველესი ადამიანის ნაშთებით.'
    },
    {
      name: 'შალვა ამირანაშვილის ხელოვნების მუზეუმი',
      type: 'museum',
      coords: [41.6943, 44.8016],
      description: 'ინახავს ქართული ოქრომჭედლობის, ტიხრული მინანქრისა და ხატწერის უნიკალურ ნიმუშებს, მათ შორის ხახულის ღვთისმშობლის კარედს.'
    },
    {
      name: 'ეთნოგრაფიული მუზეუმი (ღია ცის ქვეშ)',
      type: 'museum',
      coords: [41.7212, 44.7547],
      description: 'გიორგი ჩიტაიას სახელობის მუზეუმი კუს ტბის მახლობლად, სადაც საქართველოს სხვადასხვა კუთხიდან ჩამოტანილი ტრადიციული საცხოვრებელი სახლებია წარმოდგენილი.'
    },
    {
      name: 'მთაწმინდის პარკი',
      type: 'park',
      coords: [41.6950, 44.7860],
      description: 'გასართობი პარკი მთაწმინდის მთის თავზე, სადაც ფუნიკულიორით ადიან. აქედან მთელი თბილისის პანორამა იშლება.'
    },
    {
      name: 'რიყის პარკი',
      type: 'park',
      coords: [41.6930, 44.8105],
      description: 'თანამედროვე პარკი მტკვრის მარცხენა სანაპიროზე, მშვიდობის ხიდის მახლობლად. აქედან იწყება საბაგირო ნარიყალასკენ.'
    },
    {
      name: 'თბილისის ბოტანიკური ბაღი',
      type: 'park',
      coords: [41.6873, 44.8027],
      description: 'ნარიყალას ციხის ქვემოთ, წავკისისწყლის ხეობაში გაშენებული ბაღი, რომელიც 1845 წლიდან ფუნქციონირებს. ცნობილია ჩანჩქერით და მრავალფეროვანი ფლორით.'
    }
  ];

  // 5. Элементы правой колонки
  const titleEl = document.getElementById('info-title');
  const categoryEl = document.getElementById('info-category');
  const descEl = document.getElementById('info-description');

  let activeMarker = null;

  function showInfo(place, marker) {
    titleEl.textContent = place.name;
    categoryEl.textContent = categories[place.type].label;
    categoryEl.style.background = categories[place.type].color;
    categoryEl.style.display = 'inline-block';
    descEl.textContent = place.description;
    descEl.classList.remove('placeholder');

    if (activeMarker) {
      activeMarker.getElement().classList.remove('active-marker');
    }
    marker.getElement().classList.add('active-marker');
    activeMarker = marker;
  }

  // 6. Добавление маркеров на карту
  const markers = places.map(place => {
    const marker = L.marker(place.coords, { icon: icons[place.type], title: place.name })
      .addTo(map)
      .bindPopup(`<b>${place.name}</b><br>${categories[place.type].label}`);

    marker.on('click', () => showInfo(place, marker));
    return marker;
  });

  // 7. Подгонка масштаба под все маркеры
  const group = L.featureGroup(markers);
  map.fitBounds(group.getBounds().pad(0.1));
</script>

</body>
</html>
