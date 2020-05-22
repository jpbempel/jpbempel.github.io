## Jean-Philippe Bempel's Blog

<table style="border: 0px">
  {% for post in site.posts %}
    <tr>
      <td>{{ post.date | date_to_string }}</td><td><a href="{{ post.url }}">{{ post.title }}</a></td>
    </tr>
  {% endfor %}
</table>
