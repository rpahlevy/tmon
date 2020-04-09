<script>
  import { onMount } from "svelte";

  let svg = null;
  let das_bsolo = [204, 2, 62, 47, 5];
  let geojson = {
    type: "FeatureCollection",
    features: []
  };

  let jsonready = false;
  let d3ready = false;

  onMount(() => {
    fetch_data();
  });

  function d3loaded() {
    d3ready = true;
    if (jsonready) {
      init_graphic(geojson);
    }
  }

  function fetch_data() {
    let url = "http://hidrologi.bbws-bsolo.net/get_tma";
    fetch("https://cors-anywhere.herokuapp.com/" + url)
      .then(response => response.json())
      .then(tma => format_geojson(tma));
    // format_geojson(dummy_tma)
  }

  function format_date(str) {
    var dt;
    if (str && str.length) {
      dt = new Date(str)
    } else {
      dt = new Date()
    }

    var bulan = [
      "Jan",
      "Feb",
      "Mar",
      "Apr",
      "Mei",
      "Jun",
      "Jul",
      "Ags",
      "Sep",
      "Okt",
      "Nov",
      "Des"
    ];
    return dt.getDate() + " " + bulan[dt.getMonth()] + ", " +
      dt.getHours() + ":" + (dt.getMinutes() >= 10 ? dt.getMinutes() : "0" + dt.getMinutes());
  }

  function format_geojson(tma) {
    // console.log(tma)

    // // sort by coordinates biar bisa manipulasi jarak terlalu dekat
    // tma.sort((a, b) => {
    //   let ca = a.latlng.split(",");
    //   let cb = b.latlng.split(",");
    //   return ca[0] > cb[0] ? 1 : (ca[0] < cb[0] ? -1 : 0);
    // });

    // sort by das_bsolo
    var tma_max = 0
    var tma_min = 99999999
    var temp_tma = []
    for (const id of das_bsolo) {
      // get from tma
      for (const data of tma) {
        const id_tma = parseInt(data.id)
        const nilai_tma = data.tma
        if (id_tma === id) {
          // get min & max tma
          if (nilai_tma > tma_max) { tma_max = nilai_tma }
          if (nilai_tma < tma_min) { tma_min = nilai_tma }

          temp_tma.push(data)
          break
        }
      }
    }
    tma = temp_tma
    var tma_diff = tma_max - tma_min

    console.log(`max: ${tma_max}`)
    console.log(`min: ${tma_min}`)
    console.log(`diff: ${tma_diff}`)

    // var visualisasi ketinggian
    var start_h = 70;
    var end_h = 10;
    var d_h = (start_h - end_h);

    let prevlat = null;
    let mindiff = (0.075 * 800) / window.innerWidth;
    // console.log(mindiff)
    for (const data of tma) {
      let id = parseInt(data.id);
      if (!das_bsolo.includes(id)) {
        continue;
      }

      // let name = data.name
      let coordinates = data.latlng.split(",");

      // buat jadi 1 dimensi, hanya pakai lat
      coordinates[0] = parseFloat(coordinates[0]);
      coordinates[1] = 0;

      // manipulate lat
      let difflat = 0;
      if (prevlat) {
        difflat = Math.abs(prevlat - coordinates[0]);
        if (difflat < mindiff) {
          difflat = mindiff - difflat;
          coordinates[0] += difflat;
        }
      }
      prevlat = coordinates[0];

      let h_tma = end_h + (data.tma - tma_min) * d_h / tma_diff
      // console.log(`${h_tma} = (${data.tma} - ${tma_min}) * ${d_h} / ${tma_diff}`)
      // console.log(id +"__"+data.name + " : " + coordinates + " -> " + difflat);
      // console.log(data.status_tma + " : " + data.tma +"/"+ tma_min +" = "+ h_tma);
      // console.log(data.sampling);

      let feature = {
        type: "Feature",
        properties: {
          name: data.name,// + Math.random() * 10,
          status: data.status_tma,
          tma: data.tma.toFixed(2),
          sampling: format_date(data.sampling),
          siaga1: data.normal ? data.normal.toFixed(2) : '--',
          siaga2: data.siaga1 ? data.siaga1.toFixed(2) : '--',
          siaga3: data.siaga2 ? data.siaga2.toFixed(2) : '--',
          h_tma: h_tma
        },
        geometry: {
          type: "Point",
          coordinates: coordinates
        }
      };

      geojson.features.push(feature);
    }

    jsonready = true;
    if (d3ready) {
      init_graphic(geojson);
    }
  }

  function init_graphic(geojson) {
    // console.log('init')
    // console.log(geojson)

    var loading = document.getElementById("loading");
    if (loading) {
      loading.remove();
    }

    var now = format_date();
    var update_time = document.getElementById("update-time");
    update_time.textContent = `Update: ${now}`;

    var width = window.innerWidth;
    var height = 240;
    var margin_x = 50;
    var margin_y = 20;

    var projection = d3.geoEquirectangular();
    projection.fitExtent(
      [[margin_x, margin_y], [width - margin_x * 2, height - margin_y * 2]],
      geojson
    );
  
    if (svg) {
      d3.selectAll("g > *").remove()
      svg.remove()
    }

    svg = d3
      .selectAll("#target")
      .append("svg")

    svg.attr("width", width)
      .attr("height", height);

    var tma_line_temp = {x1:'',y1:'',x2:'',y2:''}
    var tma_line = []
    for (var feature of geojson.features) {
      var ll = feature.geometry.coordinates;
      ll = projection(ll);

      var info = {
        name: feature.properties.name,
        status: feature.properties.status,
        tma: feature.properties.tma,
        sampling: feature.properties.sampling,
        siaga1: feature.properties.siaga1,
        siaga2: feature.properties.siaga2,
        siaga3: feature.properties.siaga3,
        h_tma: feature.properties.h_tma
      };
      add_marker(ll, info);

      var tma_line_x = ll[0]
      var tma_line_y = ll[1] - feature.properties.h_tma - 30

      tma_line_temp.x2 = tma_line_temp.x1
      tma_line_temp.y2 = tma_line_temp.y1
      tma_line_temp.x1 = tma_line_x
      tma_line_temp.y1 = tma_line_y
      if (tma_line_temp.x2 !== '' && tma_line_temp.y2 !== '') {
        // clone pakai JSON.parse x stringify biar nggak kesimpen reference
        tma_line.push(JSON.parse(JSON.stringify(tma_line_temp)))
      }
    }

    // console.log(tma_line)
    svg.selectAll(null)
      .data(tma_line)
      .enter()
      .append('line')
      .attr('x1', d => d.x1)
      .attr('y1', d => d.y1)
      .attr('x2', d => d.x2)
      .attr('y2', d => d.y2)
      .style("stroke", 'lime')
      .style("stroke-width", 1)

    setTimeout(() => {
      window.location.reload()
    }, 5 * 60 * 1000);
  }

  function add_marker(ll, info) {
    if (svg == null) {
      return;
    }

    var x = ll[0];
    var y = 80;
    var h_tma = info.h_tma;

    var g = svg.append("g").attr("transform", `translate(${x}, ${y})`);

    add_tiang(g, y, h_tma);
    add_info(g, info);

    // lokasi
    // g.append("div")
    // 	.attr("y", y + 15)//magic number here
    // 	.attr("x", x)
    // 	.text(name)
  }

  function add_info(g, info, width = 65, height = 65) {
    g.attr("font-size", "7px")

    g.append("rect")
      .attr("x", -width / 2)
      .attr("y", 10)
      .attr("width", width)
      .attr("height", height)
      .attr("fill", "transparent")
      .attr("stroke", "gray");

    // info
    var fill = info.status == "siaga1" ? "lime" :
        (info.status == "siaga2" ? "yellow" : "orangered");
    g.append("circle")
      .attr("cx", -width / 2 + 5)
      .attr("cy", 17)
      .attr("r", 3)
      .style("fill", fill);

    g.append("text")
      .attr("y", 20) //magic number here
      .attr("x", -width / 2 + 9)
      .attr("font-size", "9px")
      .attr("font-weight", "bold")
      .text(info.name);

    g.append("text")
      .attr("y", 30) //magic number here
      .attr("x", -width / 2 + 9)
      .text(info.tma + " m");

    g.append("text")
      .attr("y", 40) //magic number here
      .attr("x", -width / 2 + 9)
      .text(info.sampling);

    g.append("text")
      .attr("y", 50) //magic number here
      .attr("x", -width / 2 + 9)
      .attr("color", "green")
      .text(`SH: ${info.siaga1}`);

    g.append("text")
      .attr("y", 60) //magic number here
      .attr("x", -width / 2 + 9)
      .attr("color", "orange")
      .text(`SK: ${info.siaga2}`);

    g.append("text")
      .attr("y", 70) //magic number here
      .attr("x", -width / 2 + 9)
      .attr("color", "red")
      .text(`SM: ${info.siaga3}`);
  }

  function add_tiang(g, height = 80, h_tma = 0) {
    var stroke = "gray";
    var stroke_width = 3;

    var xh_1 = -15;
    var xh_2 = +15;
    var xh2_1 = -10;
    var xh2_2 = 10;
    var yh_1 = -57;
    var yh_2 = -50;
    var yh_3 = -43;

    // hanya untuk grouping
    var tiang = g.append("g").attr("class", "tiang");

    // main
    tiang
      .append("line")
      .attr("stroke", stroke)
      .attr("stroke-width", stroke_width)
      .attr("x1", 0)
      .attr("x2", 0)
      .attr("y1", -height)
      .attr("y2", 0);

    // tma
    tiang
      .append("line")
      .attr("stroke", 'lime')
      .attr("stroke-width", 5)
      .attr("x1", -6)
      .attr("x2", 6)
      .attr("y1", -h_tma)
      .attr("y2", -h_tma);

    // // top
    // tiang
    //   .append("line")
    //   .attr("stroke", stroke)
    //   .attr("stroke-width", stroke_width)
    //   .attr("x1", xh_1)
    //   .attr("x2", xh_2)
    //   .attr("y1", yh_1)
    //   .attr("y2", yh_1);

    // // mid
    // tiang
    //   .append("line")
    //   .attr("stroke", stroke)
    //   .attr("stroke-width", stroke_width)
    //   .attr("x1", xh2_1)
    //   .attr("x2", xh2_2)
    //   .attr("y1", yh_2)
    //   .attr("y2", yh_2);

    // // bot
    // tiang
    //   .append("line")
    //   .attr("stroke", stroke)
    //   .attr("stroke-width", stroke_width)
    //   .attr("x1", xh_1)
    //   .attr("x2", xh_2)
    //   .attr("y1", yh_3)
    //   .attr("y2", yh_3);
  }
</script>

<style>
  body {
    padding: 0;
    margin: 0;
  }

  header,
  #loading {
    text-align: center;
  }

  header {
    margin-bottom: 1rem;
    padding: 1rem 0;
  }

  h1, p {
    margin: 0;
    padding: 0;
  }

  h1 {
    margin-bottom: .25rem
  }
</style>

<svelte:head>
  <script src="https://d3js.org/d3.v5.min.js" on:load={d3loaded}>

  </script>
</svelte:head>

<header>
  <h1>DAS Bengawan Solo</h1>
  <p id="update-time">Update: -</p>
</header>

<div id="loading">Loading...</div>

<div id="target"></div>
