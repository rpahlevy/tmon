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
    var dt = new Date(str);
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

    // sort by coordinates biar bisa manipulasi jarak terlalu dekat
    tma.sort((a, b) => {
      let ca = a.latlng.split(",");
      let cb = b.latlng.split(",");
      return ca[0] > cb[0] ? 1 : (ca[0] < cb[0] ? -1 : 0);
    });

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
      if (prevlat) {
        let difflat = Math.abs(prevlat - coordinates[0]);
        if (difflat < mindiff) {
          difflat = mindiff - difflat;
          coordinates[0] += difflat;
        }
      }
      prevlat = coordinates[0];

      console.log(data.name + " : " + coordinates);
      // console.log(data.status_tma + " : " + data.tma);
      // console.log(data.sampling);

      let feature = {
        type: "Feature",
        properties: {
          name: data.name,// + Math.random() * 10,
          status: data.status_tma,
          tma: data.tma.toFixed(2),
          sampling: format_date(data.sampling)
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

    for (var feature of geojson.features) {
      var ll = feature.geometry.coordinates;
      ll = projection(ll);

      var info = {
        name: feature.properties.name,
        status: feature.properties.status,
        tma: feature.properties.tma,
        sampling: feature.properties.sampling
      };
      add_marker(ll, info);
    }

    setTimeout(() => {
      window.location.reload()
    }, 5 * 60 * 1000);
  }

  function add_marker(ll, info) {
    if (svg == null) {
      return;
    }

    var x = ll[0];
    var y = 70;

    var g = svg.append("g").attr("transform", `translate(${x}, ${y})`);

    add_tiang(g, y);
    add_info(g, info);

    // lokasi
    // g.append("div")
    // 	.attr("y", y + 15)//magic number here
    // 	.attr("x", x)
    // 	.text(name)
  }

  function add_info(g, info, width = 65, height = 35) {
    g.append("rect")
      .attr("x", -width / 2)
      .attr("y", 10)
      .attr("width", width)
      .attr("height", height)
      .attr("fill", "transparent")
      .attr("stroke", "gray");

    // info
    var fill =
      info.status == "siaga1"
        ? "green"
        : info.status == "siaga2"
        ? "yellow"
        : "red";
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
      .attr("font-size", "7px")
      // .attr("font-weight", "bold")
      .text(info.tma + " m");

    g.append("text")
      .attr("y", 40) //magic number here
      .attr("x", -width / 2 + 9)
      .attr("font-size", "7px")
      // .attr("font-weight", "bold")
      .text(info.sampling);
  }

  function add_tiang(g, height = 70) {
    var stroke = "gray";
    var stroke_width = 2;

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

<svelte:head>
  <script src="https://d3js.org/d3.v5.min.js" on:load={d3loaded}>

  </script>
</svelte:head>
<center id="loading">Loading...</center>
<div id="target"></div>
