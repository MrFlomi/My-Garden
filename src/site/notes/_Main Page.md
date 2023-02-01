```dataviewjs

dv.span("** D&D Session **")
const calendarData = {
colors: {
	blue: ["#DB0F07", "#64AD62"]
	},
	intensityScaleStart: 0, 
    intensityScaleEnd: 1,
	entries: [],
}
for(let page of dv.pages('"Wicked and Divine/Sessions"')){
	calendarData.entries.push({
		date: page.date,
		intensity: page.happend,
		color: "blue",
		content: await dv.span(`[[${page.file.name}|]]`), //for hover preview
		})
}
renderHeatmapCalendar(this.container, calendarData)

```