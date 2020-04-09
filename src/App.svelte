<script>
  import { onMount } from "svelte";

  let svg = null;
  // let das_bsolo = [204, 2, 62, 47, 5];
  // let das_bsolo = [204, 2, 12, 47, 5, 54, 240, 45];
  let das_bsolo = [
    {
      id: 204,
      name: 'Colo',
      d: 0,
    },
    {
      id: 2,
      name: 'Jurug',
      d: 39,
    },
    {
      id: 12,
      name: 'Kajangan',
      d: 89, // 39 (kedungupit) + 50
    },
    {
      id: 47,
      name: 'Napel',
      d: 53, // 51 (ngawi junction) + 2
    },
    {
      id: 5,
      name: 'Cepu',
      d: 62,
    },
    {
      id: 54,
      name: 'Bojonegoro',
      d: 66,
    },
    {
      id: 240,
      name: 'Widang',
      d: 60,
    },
    {
      id: 45,
      name: 'Karanggeneng',
      d: 55,
    },
  ]
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
    // sort by das_bsolo
    var tma_max = 0
    var tma_min = 99999999
    var temp_tma = []
    for (const das of das_bsolo) {
      // get from tma
      for (const data of tma) {
        const id_tma = parseInt(data.id)
        const nilai_tma = data.tma
        if (id_tma === das.id) {
          // get min & max tma
          if (nilai_tma > tma_max) { tma_max = nilai_tma }
          if (nilai_tma < tma_min) { tma_min = nilai_tma }

          // set status
          if (nilai_tma >= data.siaga2) {
            data.status_tma = 'sm'
          } else if (nilai_tma >= data.siaga1) {
            data.status_tma = 'sk'
          } else if (nilai_tma >= data.normal) {
            data.status_tma = 'sh'
          } else {
            data.status_tma = ''
          }

          data.name = das.name
          data.d = das.d
          temp_tma.push(data)
          break
        }
      }
    }
    tma = temp_tma
    var tma_diff = tma_max - tma_min

    // console.log(`max: ${tma_max}`)
    // console.log(`min: ${tma_min}`)
    // console.log(`diff: ${tma_diff}`)

    // var visualisasi ketinggian
    var start_h = 70;
    var end_h = 10;
    var d_h = (start_h - end_h);

    let prevcoordinate = null;
    let prevlat = null;
    let mindiff = (0.5 * 800) / window.innerWidth;
    for (const data of tma) {
      let id = parseInt(data.id);

      let coordinates = data.latlng.split(",");
      coordinates[0] = parseFloat(coordinates[0]);
      coordinates[1] = parseFloat(coordinates[1]);
      // console.log(id +"__"+data.name + " : " + coordinates);

      let d = data.d * 1000
      if (prevcoordinate) {
        // rumus untuk menambahkan jarak ke koordinat
        coordinates[0] = prevcoordinate[0] + (180/Math.PI)*(d/6378137)/Math.cos(Math.PI/180*prevcoordinate[0])
      }
      prevcoordinate = coordinates;

      // buat jadi 1 dimensi, hanya pakai lat
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
      // console.log(id +"__"+data.name + " : " + coordinates + " -> " + difflat + ' -> '+ hv);
      // console.log(data.status_tma + " : " + data.tma +"/"+ tma_min +" = "+ h_tma);
      // console.log(data.sampling);

      let feature = {
        type: "Feature",
        properties: {
          name: data.name,
          status: data.status_tma,
          tma: data.tma.toFixed(2),
          sampling: format_date(data.sampling),
          sh: data.normal ? data.normal.toFixed(2) : '--',
          sk: data.siaga1 ? data.siaga1.toFixed(2) : '--',
          sm: data.siaga2 ? data.siaga2.toFixed(2) : '--',
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

    // build semua info
    var tma_line_temp = {x1:'',y1:'',x2:'',y2:''}
    var tma_line = []
    var infos = []
    for (var feature of geojson.features) {
      var ll = feature.geometry.coordinates;
      ll = projection(ll);

      var info = {
        name: feature.properties.name,
        ll: ll,
        status: feature.properties.status,
        tma: feature.properties.tma,
        sampling: feature.properties.sampling,
        sh: feature.properties.sh,
        sk: feature.properties.sk,
        sm: feature.properties.sm,
        h_tma: feature.properties.h_tma
      };
      infos.push(info)

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

    // gambar line dulu agar nantinya ada di bawah tiang & tanda
    svg.selectAll(null)
      .data(tma_line)
      .enter()
      .append('line')
        .attr('x1', d => d.x1)
        .attr('y1', d => d.y1)
        .attr('x2', d => d.x2)
        .attr('y2', d => d.y2)
        .style("stroke", '#666')
        .style("stroke-width", 1)

    // baru gambar markernya
    for (const info of infos) {
      add_marker(info)
    }

    setTimeout(() => {
      window.location.reload()
    }, 5 * 60 * 1000);
  }

  function add_marker(info) {
    if (svg == null) {
      return;
    }

    var ll = info.ll
    var x = ll[0];
    var y = 80;
    var h_tma = info.h_tma;

    var g = svg.append("g").attr("transform", `translate(${x}, ${y})`);

    add_tiang(g, y, h_tma);
    add_info(g, info);
  }

  function add_info(g, info, width = 75, height = 65) {
    var fill = "#666";
    switch (info.status) {
      case "sh":
        fill = "lime";
        break;

      case "sk":
        fill = "yellow";
        break;

      case "sm":
        fill = "red";
        break;
    }

    g.attr("font-size", "7px")

    // tma
    g.append("line")
      .attr("stroke", fill)
      .attr("stroke-width", 5)
      .attr("x1", -6)
      .attr("x2", 6)
      .attr("y1", -info.h_tma)
      .attr("y2", -info.h_tma);

    g.append("rect")
      .attr("x", -width / 2)
      .attr("y", 10)
      .attr("width", width)
      .attr("height", height)
      .attr("fill", "transparent")
      .attr("stroke", "gray");

    // info
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
      .text(`SH: ${info.sh}`);

    g.append("text")
      .attr("y", 60) //magic number here
      .attr("x", -width / 2 + 9)
      .attr("color", "orange")
      .text(`SK: ${info.sk}`);

    g.append("text")
      .attr("y", 70) //magic number here
      .attr("x", -width / 2 + 9)
      .attr("color", "red")
      .text(`SM: ${info.sm}`);
  }

  function add_tiang(g, height = 80) {
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
